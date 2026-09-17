# BÀI TẬP 5: TRADE-OFF – CIRCUIT BREAKER HAY RETRY PATTERN?

## 1. Bối cảnh bài toán

Trong hệ thống Microservices, `Order-Service` cần gọi sang hệ thống vận chuyển GHTK để tạo vận đơn.

Hệ thống GHTK có đặc điểm:

- Thỉnh thoảng xảy ra lỗi mạng.
- Một số request bị `Timeout`.
- Tuy nhiên, đây thường là lỗi tạm thời (`Transient Failure`).
- Nếu gọi lại request ngay sau đó, request có khả năng thành công.

Ví dụ:

```text
Order-Service
     |
     | Request tạo vận đơn
     v
    GHTK
     |
     X---- Timeout
     
Order-Service thử lại
     |
     | Request lần 2
     v
    GHTK
     |
     ✓---- Thành công
