# JWT và luồng Access Token Refresh Token

Tài liệu giải thích cấu trúc JWT, cách xác thực request trong Spring Security và vòng đời đăng nhập, làm mới token, đăng xuất. Được chuyển từ `Document Java (1).docx`; các ví dụ trong chín ảnh đã được chuyển thành code và HTTP có thể chỉnh sửa.

Đọc [kiến trúc Spring Security](DocumentSpringSecurity.md) trước để hiểu authentication, authorization và filter chain. Đây là tài liệu lý thuyết; các endpoint `/auth/*` là ví dụ do ứng dụng triển khai, không tự xuất hiện khi bật Resource Server.

## Mục lục

1. [JWT là gì](#1-jwt-là-gì)
2. [Cấu trúc JWT](#2-cấu-trúc-jwt)
3. [JWT trong Spring Security](#3-jwt-trong-spring-security)
4. [Ưu điểm và giới hạn](#4-ưu-điểm-và-giới-hạn)
5. [Access token và refresh token](#5-access-token-và-refresh-token)
6. [Luồng đăng nhập và làm mới token](#6-luồng-đăng-nhập-và-làm-mới-token)
7. [Các điểm đã chỉnh từ bản Word](#7-các-điểm-đã-chỉnh-từ-bản-word)
8. [Nguồn đối chiếu](#8-nguồn-đối-chiếu)

## 1 JWT là gì

JWT (JSON Web Token) là định dạng biểu diễn một tập hợp claims dưới dạng gọn để truyền giữa các bên. JWT có thể được ký hoặc được bảo vệ bằng mã hóa; tài liệu này tập trung vào JWT được ký theo dạng JWS compact thường dùng làm access token.

JWT được ký không mặc nhiên giữ bí mật dữ liệu. Base64URL là cách biểu diễn dữ liệu, không phải mã hóa bảo mật. Người giữ token có thể đọc header và payload; không đưa mật khẩu hoặc bí mật vào đó.

Luồng thường gặp:

```text
Client giữ access token
    → Gửi token trong mỗi request
    → API xác minh token
    → Kiểm tra quyền
    → Xử lý nghiệp vụ
```

JWT có thể dùng cho REST API, ứng dụng mobile và microservices. Stateless là lựa chọn kiến trúc của ứng dụng: dùng JWT không tự loại bỏ session, và hệ thống vẫn có thể lưu refresh token hoặc thông tin thu hồi token.

## 2 Cấu trúc JWT

JWT được ký theo dạng JWS compact có ba phần, phân cách bằng dấu chấm:

```text
base64Url(header).base64Url(payload).base64Url(signature)
```

Chuỗi trong bản Word được trình bày lại trên một dòng:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJtYW5oIiwiZXhwIjoxNzM4NzY5MDQxfQ.5U9im0M4tTy7bnQz0Vq_5RkUP5MUrFZrjo6Ce6eHqX8
```

Đây là chuỗi minh họa từ tài liệu, chưa xác minh chữ ký. Payload của chuỗi này có `sub` và `exp`, không có `role`; các JSON bên dưới là những ví dụ riêng. Các thời điểm `exp` trong bản Word đã nằm trong quá khứ.

| Phần | Nội dung | Vai trò |
| --- | --- | --- |
| Header | Thuật toán và metadata | Mô tả cách bảo vệ token. |
| Payload | Claims | Mang thông tin chủ thể và điều kiện sử dụng. |
| Signature | Chữ ký hoặc mã xác thực | Giúp kiểm tra tính toàn vẹn và nguồn phát hành khi dùng khóa tin cậy. |

### 2.1 Header

Ví dụ từ ảnh trong Word:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg`: thuật toán; HS256 sử dụng HMAC SHA-256 với khóa bí mật dùng chung.
- `typ`: loại token, trong ví dụ là JWT.

Header có thể giống nhau giữa nhiều token, nhưng cũng có thể thay đổi, chẳng hạn khi đổi thuật toán hoặc `kid` để nhận diện khóa. Backend phải giới hạn thuật toán chấp nhận, không tin thuật toán chỉ vì token tự khai báo.

### 2.2 Payload và claims

Ví dụ từ ảnh trong Word:

```json
{
  "sub": "duymanh",
  "role": "ROLE_ADMIN",
  "exp": 1738584658
}
```

| Claim | Ý nghĩa |
| --- | --- |
| `sub` | Chủ thể của token; có thể là ID người dùng, không bắt buộc là username. |
| `iss` | Bên phát hành token. |
| `aud` | Bên nhận được phép sử dụng token. |
| `exp` | Thời điểm hết hạn tính theo giây từ Unix epoch. |
| `nbf` | Thời điểm sớm nhất được chấp nhận. |
| `iat` | Thời điểm phát hành. |
| `jti` | Định danh token. |
| `role` | Claim tùy biến của ứng dụng, không phải claim chuẩn để tự động phân quyền. |

Ứng dụng phải thống nhất cách ánh xạ role thành authorities. Đọc được `ROLE_ADMIN` trong payload chưa đủ để cấp quyền: phải xác thực token trước.

### 2.3 Signature

Công thức minh họa trong ảnh:

```text
HMACSHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secretKey
)
```

Kết quả HMAC được Base64URL encode để tạo phần signature trong token. Nếu sửa header hoặc payload, phép kiểm tra chữ ký bằng khóa tin cậy sẽ thất bại. Công thức này chỉ áp dụng cho HS256; các thuật toán bất đối xứng dùng private key để ký và public key để kiểm tra.

## 3 JWT trong Spring Security

### 3.1 Tách vai trò cấp token và kiểm tra token

Hệ thống cấp token xác thực thông tin đăng nhập và phát hành token. API nhận token kiểm tra tính hợp lệ rồi phân quyền. Hai vai trò có thể nằm trong cùng ứng dụng hoặc ở những dịch vụ khác nhau.

Spring Security có OAuth2 Resource Server để kiểm tra JWT. Luồng tích hợp sẵn:

```text
Authorization: Bearer <access-token>
    → BearerTokenAuthenticationFilter
    → AuthenticationManager / ProviderManager
    → JwtAuthenticationProvider
    → JwtDecoder kiểm tra token
    → JwtAuthenticationConverter ánh xạ authorities
    → JwtAuthenticationToken
    → SecurityContext
    → Phân quyền
    → Controller
```

Nguồn: [Spring Security JWT Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html). Custom `JwtAuthenticationFilter` là một phương án tự triển khai; không bắt buộc đặt filter tự viết trước `UsernamePasswordAuthenticationFilter` khi dùng cơ chế tích hợp sẵn.

### 3.2 Những điều cần kiểm tra

- Thuật toán được phép và chữ ký bằng khóa tin cậy.
- Hạn sử dụng `exp`, thời điểm `nbf` nếu có, với độ lệch đồng hồ được kiểm soát.
- Issuer `iss` và audience `aud` phù hợp với API theo cấu hình.
- Loại token và các claims bắt buộc theo hợp đồng phát hành.
- Trạng thái thu hồi hoặc trạng thái tài khoản nếu ứng dụng có chính sách kiểm tra thêm.

Decode payload không tương đương validate token. Không dùng refresh token để gọi API nghiệp vụ.

## 4 Ưu điểm và giới hạn

### Ưu điểm

- API có thể xác minh token mà không tải session tập trung cho mỗi request.
- Các service có thể dùng chung hệ thống định danh và kiểm tra token theo audience phù hợp.
- Claims giúp truyền thông tin cần thiết theo một hợp đồng thống nhất.

### Giới hạn

- Token bị đánh cắp có thể bị sử dụng đến khi hết hạn hoặc bị thu hồi.
- Lộ khóa ký có thể cho phép giả mạo token; cần quản lý và luân chuyển khóa.
- Claims có thể cũ sau khi quyền hoặc trạng thái tài khoản thay đổi.
- Thu hồi ngay thường cần thêm trạng thái phía server, ví dụ denylist theo `jti` hoặc phiên bản token.
- Token có thể lớn hơn một định danh session ngắn.

Header `Authorization` là cách gửi access token trong các ví dụ này. Tránh đưa token vào query string vì có thể lọt vào lịch sử và log. Nếu dùng cookie, phải xem xét CSRF và chính sách cookie.

## 5 Access token và refresh token

| Đặc điểm | Access token | Refresh token |
| --- | --- | --- |
| Mục đích | Truy cập API nghiệp vụ. | Xin access token mới. |
| Thời hạn trong ví dụ Word | 15 phút. | 7 ngày. |
| Nơi gửi | API qua Bearer header. | Endpoint làm mới token. |
| Định dạng | Có thể là JWT hoặc opaque token. | Có thể là opaque token hoặc JWT. |
| Lưu phía backend | Có thể chỉ xác minh chữ ký, tùy chính sách thu hồi. | Nên quản lý trạng thái để thu hồi và phát hiện sử dụng lại. |

15 phút và 7 ngày là lựa chọn minh họa, không phải thời hạn mặc định của Spring Security.

### Lưu token trên trình duyệt

Bản Word dùng access token trong `localStorage` và refresh token trong cookie HttpOnly. `localStorage` có thể bị JavaScript đọc nên có nguy cơ lộ token khi có XSS. Có thể giữ access token trong bộ nhớ hoặc dùng kiến trúc BFF/session tùy ứng dụng.

Cookie refresh token nên cấu hình `HttpOnly`, `Secure`, `SameSite` phù hợp, giới hạn path và có chính sách chống CSRF cho endpoint nhận cookie. HttpOnly hạn chế JavaScript đọc cookie nhưng không ngăn trình duyệt gửi cookie trong request.

Lưu refresh token ở backend và gửi cookie cho client là hai việc có thể thực hiện đồng thời. Một phương án là lưu giá trị băm token cùng user, thời hạn, phiên/thiết bị và trạng thái thu hồi.

## 6 Luồng đăng nhập và làm mới token

### 6.1 Đăng nhập

Ví dụ từ ảnh, bổ sung HTTP headers; mật khẩu chỉ là dữ liệu minh họa:

```http
POST /auth/login HTTP/1.1
Content-Type: application/json

{
  "username": "manh",
  "password": "123"
}
```

Backend xác thực username/password, tạo access token và refresh token, lưu thông tin refresh token rồi gửi refresh token trong cookie. Body từ ảnh:

```json
{
  "accessToken": "xxx"
}
```

Nếu gửi refresh token bằng cookie, response đồng thời có `Set-Cookie`; thuộc tính và thời hạn phải đồng bộ với chính sách của backend.

### 6.2 Gọi API

Header từ ảnh:

```http
Authorization: Bearer <access-token>
```

API xác minh token, tạo Authentication, kiểm tra quyền và xử lý request. Khi cấu hình stateless, mỗi request phải mang thông tin xác thực hợp lệ.

### 6.3 Access token hết hạn

Ảnh Word minh họa:

```text
401 Unauthorized - Token expired
```

Với API Bearer, access token hết hạn thường dẫn đến HTTP 401. Phần thông báo cụ thể phụ thuộc handler; không mặc định trả đúng chuỗi trên. 403 thường thể hiện thiếu quyền sau xác thực. Client không nên tự refresh vô hạn cho mọi lỗi 401.

### 6.4 Làm mới token

Request từ ảnh:

```http
POST /auth/refresh HTTP/1.1
Cookie: refreshToken=abc...
```

Backend kiểm tra refresh token, hạn sử dụng, trạng thái thu hồi và phiên liên quan. Khi hợp lệ, cấp access token mới và có thể xoay vòng refresh token: vô hiệu hóa token cũ, tạo token mới trong cùng một giao dịch phù hợp. Nếu phát hiện dùng lại token cũ, có thể thu hồi cả nhóm token của phiên đó.

Body từ ảnh:

```json
{
  "accessToken": "new-token"
}
```

Nếu xoay vòng refresh token, response cũng cập nhật cookie bằng `Set-Cookie`. Client cập nhật access token và thử lại request bị lỗi; khi refresh thất bại, yêu cầu đăng nhập lại.

Trong OAuth2, endpoint token chuẩn sử dụng hợp đồng khác; `/auth/refresh` ở đây chỉ minh họa API tùy biến. Theo [RFC 9700 mục 4.14](https://www.rfc-editor.org/rfc/rfc9700.html#section-4.14), refresh token cho public client phải được ràng buộc với bên gửi hoặc sử dụng rotation để phát hiện replay.

### 6.5 Đăng xuất

Backend thu hồi refresh token hoặc phiên tương ứng và xóa cookie bằng cùng path/domain đã dùng khi tạo. Client xóa access token đang giữ. Chỉ xóa cookie không vô hiệu hóa bản sao token đã bị đánh cắp.

Access token đã phát hành vẫn có thể dùng đến hết hạn nếu API chỉ kiểm tra chữ ký và thời hạn. Nếu cần logout có hiệu lực ngay, bổ sung chính sách thu hồi và kiểm tra phía API.

```text
Đăng nhập
    → Access token + refresh token
    → Gọi API bằng access token
    → Access token hết hạn
    → Gọi endpoint refresh
        → Hợp lệ: cấp access token mới và xoay vòng refresh token
        → Không hợp lệ: đăng nhập lại
    → Logout: thu hồi refresh token/phiên và xóa token phía client
```

## 7 Các điểm đã chỉnh từ bản Word

| Cách diễn đạt ban đầu | Cách hiểu đã chỉnh |
| --- | --- |
| JWT là chuỗi mã hóa JSON. | JWT là định dạng claims; JWT được ký không mặc nhiên mã hóa payload. |
| JWT luôn có ba phần. | Ba phần là dạng JWS compact được trình bày ở đây; JWE compact có năm phần. |
| Dùng JWT thì không cần session. | Stateless phụ thuộc cấu hình và thiết kế ứng dụng. |
| Chỉ cần đọc JWT và kiểm tra chữ ký. | Cần kiểm tra cả thời hạn, issuer, audience và chính sách token. |
| Spring luôn dùng custom JWT filter. | Resource Server đã có cơ chế xác thực JWT tích hợp sẵn. |
| Không thể thu hồi token. | Có thể thu hồi bằng trạng thái bổ sung; token tự chứa không tự biết bị thu hồi. |
| Lưu refresh token vào DB hoặc cookie. | Backend có thể quản lý token đồng thời client giữ cookie. |
| Xóa cookie là logout hoàn toàn. | Cần thu hồi refresh token; access token còn hiệu lực tùy chính sách. |

## 8 Nguồn đối chiếu

- [RFC 7519 về JWT](https://www.rfc-editor.org/rfc/rfc7519.html): định dạng và registered claims.
- [RFC 7515 về JWS](https://www.rfc-editor.org/rfc/rfc7515.html): biểu diễn compact và chữ ký.
- [RFC 7516 về JWE](https://www.rfc-editor.org/rfc/rfc7516.html): JWT được bảo vệ bằng mã hóa.
- [Spring Security JWT Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html): kiểm tra JWT và ánh xạ authorities.
- [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html): bảo vệ refresh token và rotation.
