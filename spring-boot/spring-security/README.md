# Spring Security

Nhóm tài liệu về xác thực và phân quyền trong ứng dụng Spring.

## Thứ tự đọc

| Thứ tự | Tài liệu | Nội dung |
| --- | --- | --- |
| 1 | [Spring Security](DocumentSpringSecurity.md) | Kiến trúc, filter chain, authentication, authorization, JWT và OAuth2/OIDC ở mức tổng quan. |
| 2 | [JWT và Access Token Refresh Token](DocumentJWT.md) | JWT và claims, chữ ký, xác thực request, refresh token, rotation và logout. |
| 3 | [Resource Server](DocumentResourceServer.md) | Bảo vệ API, JWT/opaque token, issuer, audience và scope. |
| 4 | [OpenID Connect (OIDC)](DocumentOIDC.md) | Đăng nhập, ID token, callback, session và kết hợp với Resource Server. |
| 5 | [Authorization Code + PKCE](DocumentAuthorizationCodePKCE.md) | Flow đổi code, verifier/challenge, state/nonce, ví dụ HTTP và tích hợp Spring Security. |
| 6 | [Spring Authorization Server](DocumentSpringAuthorizationServer.md) | Bên cấp token, endpoint, client, filter chain, cấu hình Boot, thực hành và vận hành. |

Đọc phần tổng quan trước, tiếp đến JWT, Resource Server, OIDC, Authorization Code + PKCE và Spring Authorization Server. Ví dụ trong các tài liệu là minh họa riêng lẻ, chưa phải ứng dụng hoàn chỉnh.

## Tra cứu flow

| Tài liệu | Flow minh họa bằng Mermaid |
| --- | --- |
| [Spring Security](DocumentSpringSecurity.md#5-luồng-xử-lý-request) | Request qua filter chain; form login; session và JWT stateless; phân quyền HTTP và method. |
| [JWT](DocumentJWT.md#6-luồng-đăng-nhập-và-làm-mới-token) | Cấp token; refresh và rotation; logout và giới hạn thu hồi access token. |
| [Resource Server](DocumentResourceServer.md#3-logic-resource-server) | Xác thực JWT và phân quyền; opaque token introspection. |
| [OIDC](DocumentOIDC.md#3-logic-đăng-nhập-oidc) | Authorization Code với session; PKCE; chọn chain khi ghép web và API. |
| [Authorization Code + PKCE](DocumentAuthorizationCodePKCE.md#3-flow-đầy-đủ) | Toàn bộ giao dịch từ tạo verifier đến đổi code và gọi API. |
| [Spring Authorization Server](DocumentSpringAuthorizationServer.md#3-flow-cấp-token-và-gọi-api) | Đăng nhập tại bên cấp token, đổi code, nhận token và gọi Resource Server. |

Xem trên trình đọc Markdown hỗ trợ Mermaid để hiển thị sơ đồ. Mã sơ đồ có thể chỉnh sửa ngay trong tài liệu.

[Quay về mục lục Spring Boot](../README.md)
