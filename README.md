# BÀI TẬP 5: TRADE-OFF: CIRCUIT BREAKER HAY RETRY PATTERN?

## 1. Bối cảnh bài toán

Trong hệ thống Microservice, **Order-Service** cần gọi sang hệ thống của đơn vị vận chuyển **GHTK** để tạo vận đơn.

Hệ thống GHTK có đặc thù là thường xuyên xảy ra tình trạng **mạng chập chờn (Network Glitch)**. Thỉnh thoảng một request gọi sang GHTK có thể bị **Timeout**, nhưng nếu thực hiện request lại ngay sau đó thì request có thể thành công.

Đây là dạng lỗi **Transient Failure**, tức là lỗi xảy ra tạm thời và có khả năng tự biến mất sau một khoảng thời gian ngắn.

Ví dụ:

```text
Order-Service
     |
     |--- Request 1 ---> GHTK
     |                    |
     |                  Timeout
     |
     |--- Request 2 ---> GHTK
     |                    |
     |                  Success
```

Với tình huống này, System Architect cần lựa chọn cơ chế phù hợp để bảo vệ Order-Service và tăng khả năng xử lý thành công request.

Hai giải pháp được xem xét là:

1. **Circuit Breaker**
2. **Retry Pattern kết hợp Exponential Backoff**

---

# PHẦN 1 – ĐỀ XUẤT ĐA GIẢI PHÁP

## 2. Giải pháp 1: Dùng Circuit Breaker thuần túy

### 2.1. Khái niệm

**Circuit Breaker** là một Design Pattern dùng để bảo vệ hệ thống Microservice khỏi các lỗi liên tiếp và tránh xảy ra **Cascading Failure**.

Circuit Breaker thường có 3 trạng thái:

* **CLOSED**: Request được phép đi bình thường.
* **OPEN**: Hệ thống phát hiện nhiều lỗi và tạm thời chặn request đến service lỗi.
* **HALF_OPEN**: Sau một khoảng thời gian, hệ thống cho phép một số request thử lại để kiểm tra service đã hoạt động bình thường hay chưa.

Luồng hoạt động:

```text
             Request thành công
                  |
                  v
             +----------+
             |  CLOSED  |
             +----------+
                  |
             Nhiều lỗi liên tiếp
                  |
                  v
             +----------+
             |   OPEN   |
             +----------+
                  |
            Sau thời gian chờ
                  |
                  v
             +------------+
             | HALF_OPEN  |
             +------------+
               /        \
          Thành công      Thất bại
              |              |
              v              v
           CLOSED           OPEN
```

### 2.2. Áp dụng vào bài toán

Nếu Order-Service gọi GHTK và liên tục nhận Timeout, Circuit Breaker có thể chuyển sang trạng thái **OPEN**.

Khi đó, các request tiếp theo sẽ không được gửi sang GHTK trong một khoảng thời gian.

Ví dụ:

```text
Order-Service
     |
     | Request 1 ---> GHTK ---> Timeout
     |
     | Request 2 ---> GHTK ---> Timeout
     |
     | Request 3 ---> GHTK ---> Timeout
     |
     | Circuit Breaker OPEN
     |
     X Request tiếp theo bị chặn
```

### 2.3. Ưu điểm

* Bảo vệ Order-Service khi GHTK gặp sự cố kéo dài.
* Giúp ngăn chặn **Cascading Failure**.
* Giảm số lượng request gửi đến một service đang gặp sự cố.
* Giúp hệ thống nhanh chóng trả về lỗi hoặc sử dụng **Fallback** thay vì chờ Timeout quá lâu.
* Phù hợp với trường hợp service đích bị **sập hẳn hoặc lỗi liên tục**.

### 2.4. Nhược điểm

Trong trường hợp lỗi mạng chỉ là **Transient Failure**, Circuit Breaker thuần túy có thể chưa phải lựa chọn tối ưu.

Ví dụ:

```text
Request 1 ---> Timeout
Request 2 ---> Có thể thành công
Request 3 ---> Có thể thành công
```

Nếu Circuit Breaker nhanh chóng mở mạch sau một số lỗi, những request sau đó có thể bị chặn mặc dù GHTK đã hoạt động bình thường trở lại.

Do đó:

> Circuit Breaker phù hợp hơn với việc **ngăn chặn lỗi kéo dài**, thay vì cố gắng xử lý một lỗi mạng chập chờn bằng cách thử lại ngay.

---

# 3. Giải pháp 2: Retry Pattern kết hợp Exponential Backoff

## 3.1. Khái niệm

**Retry Pattern** cho phép hệ thống tự động thực hiện lại request khi request trước đó thất bại do một số lỗi có khả năng tạm thời.

Trong bài toán này, khi Order-Service gọi GHTK và gặp:

```text
TimeoutException
```

hệ thống sẽ tự động thử lại request.

Tuy nhiên, thay vì retry liên tục ngay lập tức, hệ thống sử dụng **Exponential Backoff** để tăng thời gian chờ giữa các lần retry.

Ví dụ:

```text
Lần 1 ---> Timeout
     |
     | Chờ
     v
Lần 2 ---> Timeout
     |
     | Chờ lâu hơn
     v
Lần 3 ---> Success
```

Một chiến lược Exponential Backoff phổ biến có thể là:

```text
Retry 1: chờ 1 giây
Retry 2: chờ 2 giây
Retry 3: chờ 4 giây
```

Trong bài tập này, yêu cầu cụ thể là:

```text
Retry tối đa 3 lần
Mỗi lần cách nhau 2 giây
```

nên hệ thống sẽ thực hiện retry theo khoảng thời gian cố định 2 giây.

## 3.2. Áp dụng vào bài toán

Ví dụ Order-Service gửi request tạo vận đơn:

```text
Request lần đầu
       |
       v
    GHTK
       |
    Timeout
       |
       v
 Chờ 2 giây
       |
       v
 Retry lần 1
       |
       v
    GHTK
       |
    Timeout
       |
       v
 Chờ 2 giây
       |
       v
 Retry lần 2
       |
       v
    GHTK
       |
    Success
```

Nhờ Retry Pattern, hệ thống có cơ hội xử lý thành công những lỗi mạng xảy ra tạm thời.

## 3.3. Ưu điểm

* Phù hợp với **Transient Failure**.
* Có khả năng tự động xử lý Timeout tạm thời.
* Không cần người dùng thực hiện lại request thủ công.
* Tăng khả năng request thành công khi service đích vẫn đang hoạt động nhưng mạng không ổn định.
* Có thể kết hợp với Backoff để tránh gửi request liên tục đến service đang gặp vấn đề.

## 3.4. Nhược điểm

* Nếu service đích bị sập hẳn, Retry sẽ tiếp tục tạo thêm request không cần thiết.
* Có thể làm tăng tải cho service đang gặp sự cố.
* Tăng thời gian xử lý request.
* Nếu không giới hạn số lần retry, có thể gây ra **Retry Storm**.
* Có rủi ro tạo dữ liệu trùng nếu API không có tính **Idempotency**.

---

# PHẦN 2 – SO SÁNH HAI GIẢI PHÁP

## 4. Bảng so sánh

| Tiêu chí                        | Circuit Breaker thuần túy                   | Retry + Exponential Backoff      |
| ------------------------------- | ------------------------------------------- | -------------------------------- |
| Mục đích chính                  | Ngăn hệ thống tiếp tục gọi service đang lỗi | Thử lại request khi lỗi tạm thời |
| Phù hợp với Transient Failure   | Không tối ưu nếu dùng một mình              | Phù hợp                          |
| Phù hợp với System Crash        | Rất phù hợp                                 | Không phù hợp nếu dùng một mình  |
| Xử lý Timeout tạm thời          | Có thể chặn request sau nhiều lỗi           | Có thể thử lại                   |
| Khả năng tăng tỷ lệ thành công  | Không trực tiếp tăng bằng retry             | Có                               |
| Bảo vệ khỏi Cascading Failure   | Tốt                                         | Hạn chế nếu dùng một mình        |
| Giảm request đến service lỗi    | Tốt khi Circuit mở                          | Không, vì vẫn tiếp tục retry     |
| Ảnh hưởng thời gian phản hồi    | Có thể trả lỗi/fallback nhanh khi mạch mở   | Tăng thời gian do phải chờ retry |
| Rủi ro tạo thêm tải             | Thấp khi Circuit mở                         | Có nếu retry quá nhiều           |
| Rủi ro Retry Storm              | Thấp                                        | Có                               |
| Cần giới hạn số lần retry       | Không phải cơ chế chính                     | Có                               |
| Có thể dùng Fallback            | Có                                          | Có thể kết hợp                   |
| Phù hợp với lỗi mạng chập chờn  | Hạn chế nếu dùng riêng                      | Phù hợp                          |
| Phù hợp với service sập hẳn     | Phù hợp                                     | Không nên retry liên tục         |
| Có thể kết hợp với Pattern khác | Có                                          | Có                               |

---

## 5. So sánh theo từng loại lỗi

### 5.1. Trường hợp 1: Transient Failure

Transient Failure là lỗi xảy ra trong thời gian ngắn và sau đó hệ thống có thể hoạt động trở lại.

Ví dụ:

```text
Request 1 ---> Timeout

Sau 2 giây

Request 2 ---> Success
```

Trong trường hợp này, **Retry Pattern** phù hợp vì hệ thống có thể thử lại request sau một khoảng thời gian.

Nếu chỉ sử dụng Circuit Breaker, hệ thống có thể mở mạch và từ chối các request tiếp theo, trong khi GHTK thực tế đã hoạt động trở lại.

---

### 5.2. Trường hợp 2: System Crash

System Crash là trường hợp service đích bị lỗi nghiêm trọng hoặc ngừng hoạt động.

Ví dụ:

```text
Order-Service
     |
     +----> GHTK
     |       X
     |      DOWN
     |
     +----> Retry
     |       X
     |
     +----> Retry
     |       X
     |
     +----> Retry
             X
```

Trong trường hợp này, Retry không giúp giải quyết vấn đề vì service vẫn đang bị down.

Nếu retry quá nhiều, Order-Service còn có thể tạo thêm tải không cần thiết.

Circuit Breaker phù hợp hơn trong trường hợp này:

```text
GHTK DOWN
   |
   v
Nhiều request thất bại
   |
   v
Circuit Breaker OPEN
   |
   v
Không tiếp tục gọi GHTK
```

Nhờ đó, hệ thống hạn chế việc một service lỗi làm ảnh hưởng đến các service khác.

---

# PHẦN 3 – TRIỂN KHAI GIẢI PHÁP

## 6. Chốt giải pháp

Đối với bài toán đã cho, giải pháp được lựa chọn là:

> **Retry Pattern kết hợp với Backoff**

Lý do là lỗi được mô tả là **Network Glitch / Transient Failure**.

GHTK không phải lúc nào cũng bị lỗi mà chỉ thỉnh thoảng request bị Timeout. Khi gọi lại sau đó, request có khả năng thành công.

Do đó, thay vì ngay lập tức ngắt mạch, Order-Service nên cho phép hệ thống tự động retry một số lần.

Đồng thời cần giới hạn số lần retry để tránh tạo quá nhiều request đến GHTK.

---

# 7. Cấu hình Retry bằng YAML

Ví dụ cấu hình với **Spring Boot + Resilience4j**:

```yaml
resilience4j:
  retry:
    instances:
      ghtkRetry:
        max-attempts: 3
        wait-duration: 2s
        retry-exceptions:
          - java.util.concurrent.TimeoutException
```

## 8. Giải thích cấu hình

### `resilience4j`

Đây là phần cấu hình cho thư viện Resilience4j.

### `retry`

Khai báo cơ chế Retry.

### `instances`

Cho phép tạo nhiều cấu hình Retry khác nhau cho các service hoặc nghiệp vụ khác nhau.

### `ghtkRetry`

Tên của Retry instance dành cho việc gọi GHTK.

Có thể đặt tên khác, ví dụ:

```yaml
ghtkRetry
```

hoặc:

```yaml
createShippingRetry
```

### `max-attempts: 3`

Hệ thống được phép thực hiện tối đa **3 lần thử**.

```text
Attempt 1
   |
 Timeout
   |
 2 giây
   |
Attempt 2
   |
 Timeout
   |
 2 giây
   |
Attempt 3
```

Nếu lần thứ 3 vẫn Timeout thì hệ thống sẽ dừng retry và trả lỗi.

### `wait-duration: 2s`

Thời gian chờ giữa các lần retry là **2 giây**.

### `retry-exceptions`

Chỉ thực hiện retry khi gặp exception được chỉ định.

Trong bài toán này là:

```text
java.util.concurrent.TimeoutException
```

Điều này rất quan trọng vì **không nên retry tất cả các loại exception**.

Ví dụ, nếu request bị lỗi do dữ liệu không hợp lệ thì retry không giúp giải quyết vấn đề.

---

# 9. Ví dụ sử dụng Retry trong Order-Service

Nếu sử dụng Resilience4j, có thể cấu hình annotation:

```java
@Retry(name = "ghtkRetry")
public ShippingResponse createShippingOrder(Order order) {
    return ghtkClient.createOrder(order);
}
```

Luồng xử lý:

```text
Order-Service
     |
     | createShippingOrder()
     v
 GHTK API
     |
     +---- TimeoutException
              |
              v
          Retry 1
              |
           2 giây
              |
              v
          GHTK API
              |
              +---- TimeoutException
                       |
                       v
                    Retry 2
                       |
                    2 giây
                       |
                       v
                    GHTK API
                       |
                    Success
```

Nếu cả 3 lần đều Timeout:

```text
Attempt 1 ---> Timeout
     |
   2 giây
     |
Attempt 2 ---> Timeout
     |
   2 giây
     |
Attempt 3 ---> Timeout
     |
     v
Retry kết thúc
     |
     v
Trả lỗi
```

---

# 10. BẪY DỮ LIỆU – RỦI RO KHI RETRY

## 10.1. Vấn đề

Retry có một rủi ro rất quan trọng.

Giả sử GHTK thực hiện trừ tiền tài khoản mỗi khi nhận được request.

Order-Service gửi:

```text
Request 1 ---> GHTK
```

GHTK thực tế đã nhận và xử lý request, đồng thời đã trừ tiền.

Tuy nhiên, do mạng gặp sự cố, Order-Service không nhận được response và nghĩ rằng request đã thất bại.

```text
Order-Service                 GHTK
     |                         |
     |--- Create Order ------->|
     |                         |
     |                    Trừ tiền
     |                         |
     |<---- Timeout -----------X
```

Order-Service thực hiện Retry:

```text
Order-Service                 GHTK
     |                         |
     |--- Retry ------------->|
     |                         |
     |                    Trừ tiền lần 2
     |                         |
     |<---- Success -----------|
```

Kết quả:

```text
Một đơn hàng
      |
      +---- Lần 1: Trừ tiền
      |
      +---- Retry: Trừ tiền lần 2
```

Khách hàng có thể bị **trừ tiền hai lần** dù thực tế chỉ muốn thực hiện một giao dịch.

---

# 11. Khái niệm Idempotency

**Idempotency** là tính chất đảm bảo rằng việc thực hiện cùng một request nhiều lần vẫn tạo ra **cùng một kết quả về mặt nghiệp vụ** như thực hiện một lần.

Ví dụ, Order-Service gửi:

```text
POST /shipping/orders
Idempotency-Key: ORDER-12345
```

GHTK nhận request lần đầu:

```text
ORDER-12345
     |
     v
Tạo vận đơn
     |
     v
Trừ tiền
```

Khi Order-Service retry với cùng `Idempotency-Key`:

```text
ORDER-12345
     |
     v
GHTK kiểm tra
     |
     v
Đã xử lý trước đó
     |
     v
Không tạo thêm vận đơn
Không trừ tiền lần nữa
     |
     v
Trả lại kết quả cũ
```

Như vậy, dù request được gửi nhiều lần, nghiệp vụ vẫn chỉ được thực hiện một lần.

---

# 12. Ví dụ về Idempotency Key

Order-Service có thể tạo một mã duy nhất cho mỗi yêu cầu tạo vận đơn:

```text
Idempotency-Key = ORDER-2026-000001
```

Request lần đầu:

```http
POST /api/shipping/orders

Idempotency-Key: ORDER-2026-000001
```

Nếu bị Timeout, Order-Service retry:

```http
POST /api/shipping/orders

Idempotency-Key: ORDER-2026-000001
```

GHTK sẽ kiểm tra:

```text
Idempotency-Key
       |
       v
Có tồn tại trong database/cache?
       |
   +---+---+
   |       |
  Có      Không
   |       |
   v       v
Trả lại   Xử lý
kết quả   request
cũ          |
            v
       Lưu kết quả
```

Điều này giúp tránh việc một nghiệp vụ bị thực hiện nhiều lần do Retry.

---

# 13. Kết luận

Trong bài toán Order-Service gọi GHTK, lỗi chủ yếu là **Network Glitch / Transient Failure**, tức là request đôi khi bị Timeout nhưng gọi lại có thể thành công.

Có hai giải pháp được xem xét:

### Giải pháp 1: Circuit Breaker

Phù hợp hơn khi:

* Service đích bị lỗi liên tục.
* Service bị sập.
* Cần ngăn Cascading Failure.
* Cần ngừng gửi request đến service đang gặp sự cố.

### Giải pháp 2: Retry + Backoff

Phù hợp với:

* Network Glitch.
* Timeout tạm thời.
* Service vẫn hoạt động nhưng request thỉnh thoảng thất bại.

Đối với bài toán này, sử dụng:

```text
Retry Pattern
     +
Backoff
```

với cấu hình:

```yaml
resilience4j:
  retry:
    instances:
      ghtkRetry:
        max-attempts: 3
        wait-duration: 2s
        retry-exceptions:
          - java.util.concurrent.TimeoutException
```

Tuy nhiên, Retry cần được sử dụng có kiểm soát. Hệ thống cần giới hạn số lần retry và chỉ retry những lỗi có khả năng tạm thời.

Đặc biệt, đối với các API có nghiệp vụ quan trọng như **trừ tiền, tạo đơn hàng hoặc tạo vận đơn**, cần thiết kế **Idempotency** để tránh việc Retry làm phát sinh giao dịch hoặc dữ liệu trùng.

## Tóm tắt

```text
                 Lỗi xảy ra
                     |
          +----------+----------+
          |                     |
     Lỗi tạm thời          Service sập
     (Transient)             (Crash)
          |                     |
          v                     v
   Retry + Backoff       Circuit Breaker
          |                     |
          v                     v
   Thử lại có giới hạn     Ngắt request
          |                     |
          +----------+----------+
                     |
              Idempotency
                     |
                     v
        Tránh xử lý trùng nghiệp vụ
```

**Kết luận:** Với tình huống GHTK thường xuyên gặp Timeout ngắn hạn nhưng có khả năng thành công khi gọi lại, **Retry Pattern kết hợp Backoff** là cơ chế phù hợp để xử lý lỗi tạm thời. Trong một hệ thống thực tế, Retry cũng có thể được kết hợp với Circuit Breaker để vừa xử lý Transient Failure vừa bảo vệ hệ thống khi GHTK gặp sự cố kéo dài.
