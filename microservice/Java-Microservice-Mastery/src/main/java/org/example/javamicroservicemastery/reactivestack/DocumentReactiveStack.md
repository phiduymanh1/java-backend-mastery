# Reactive Stack

1. **Reactive Stack là gì?**

* Reactive Stack là mô hình lập trình giúp ứng dụng xử lý nhiều request đồng thời theo cách:
  * Asynchronous (bất đồng bộ): Không cần chờ một tác vụ hoàn thành mới làm việc khác.
  * Non-blocking (không chặn): Thread không bị giữ lại trong lúc chờ I/O (DB, HTTP, File...).
  * Event-driven (hướng sự kiện): Khi tác vụ hoàn thành sẽ phát ra sự kiện để tiếp tục xử lý.
* Reactive Stack được triển khai bằng Spring WebFlux.

2. **WebFlux là gì?**

* Spring WebFlux là framework Reactive của Spring.
*Nó thay thế cách xử lý truyền thống của Spring MVC bằng mô hình Reactive.
* WebFlux sử dụng:
  * Reactor
  * Mono
  * Flux
  * Event Loop
  * Netty (mặc định)

3. **Vì sao Spring Cloud Gateway dùng WebFlux?**
* Vì Gateway chủ yếu nhận, kiểm tra và chuyển tiếp request
* WebFlux giúp xử lý nhiều request đồng thời với ít thread hơn nhờ cơ chế non-blocking.

4. **Các service phía sau có bắt buộc dùng WebFlux không?**
* Không. Chúng hoàn toàn có thể dùng Spring MVC. Đây cũng là kiến trúc phổ biến trong nhiều hệ thống Microservice.