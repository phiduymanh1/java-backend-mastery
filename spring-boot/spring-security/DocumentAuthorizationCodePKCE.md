# Authorization Code + PKCE

Đọc sau [OIDC](DocumentOIDC.md). Tài liệu đi từ giao thức đến cách áp dụng trong Spring Security; các URL và token dưới đây chỉ là minh họa, chưa phải ứng dụng chạy được.

## 1. Mục đích và các bên tham gia

Authorization Code là grant của OAuth2: client nhận một code ngắn hạn qua callback, rồi đổi code tại token endpoint. Code không phải access token và không dùng gọi API.

PKCE (Proof Key for Code Exchange) ràng buộc lần đổi code với một bí mật tạm thời do client tạo cho từng giao dịch. Kẻ lấy được code nhưng không có bí mật này không đổi được code trong flow được thực thi đúng.

| Bên | Vai trò |
| --- | --- |
| Người dùng / Browser | Đăng nhập tại bên định danh và đi qua các redirect. |
| OAuth2 Client | Khởi tạo yêu cầu, giữ verifier, xử lý callback và đổi code. |
| Authorization Server / OP | Xác thực, cấp code và token; kiểm tra PKCE. |
| Resource Server | Kiểm tra access token và phân quyền API. |

OAuth2 phục vụ ủy quyền. Khi dùng để đăng nhập theo OIDC, thêm scope `openid` và xác thực ID token. Xem [OAuth2 Authorization Code](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.1) và [OIDC Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth).

## 2. Các giá trị cần phân biệt

| Giá trị | Nơi gửi / giữ | Ý nghĩa |
| --- | --- | --- |
| `code_verifier` | Client giữ; gửi ở token request | Bí mật ngẫu nhiên cho một giao dịch. |
| `code_challenge` | Authorization request | Giá trị suy ra từ verifier. |
| `code_challenge_method` | Authorization request | Dùng `S256` trong tài liệu này. |
| `code` | Callback rồi token request | Mã một lần, gắn với client và redirect URI. |
| `state` | Authorization request và callback | Đối chiếu callback với giao dịch đã khởi tạo. |
| `nonce` | OIDC request và claim trong ID token | Ràng buộc ID token với yêu cầu; phải đối chiếu nếu đã gửi. |
| `client_secret` | Chỉ ở confidential client | Xác thực client, không thay thế verifier. |

Verifier dài 43–128 ký tự thuộc tập `A-Z a-z 0-9 - . _ ~`, sinh bằng nguồn ngẫu nhiên mật mã. Không tái sử dụng giữa các lần đăng nhập.

```text
code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))
```

Base64URL bỏ padding `=`, đổi `+` thành `-`, `/` thành `_`. Challenge không phải mã hóa có thể giải ngược verifier. Xem [RFC 7636, mục 4](https://www.rfc-editor.org/rfc/rfc7636.html#section-4).

## 3. Flow đầy đủ

```mermaid
sequenceDiagram
    actor B as Browser / Người dùng
    participant C as OAuth2 Client
    participant A as Authorization Server / OP
    participant R as Resource Server
    C->>C: Tạo verifier, challenge, state và nonce nếu dùng OIDC
    C->>C: Lưu giao dịch tạm thời
    C-->>B: Điều hướng tới authorization endpoint
    B->>A: response_type=code, challenge S256, state, scope
    A->>A: Xác thực người dùng và xin chấp thuận nếu cần
    A-->>B: Redirect tới callback với code và state
    B->>C: Callback
    C->>C: Kiểm tra state, lỗi callback và giao dịch còn hiệu lực
    C->>A: POST token endpoint với code, verifier, redirect_uri
    Note over C,A: Confidential client còn phải xác thực client
    A->>A: Kiểm tra code và hash verifier khớp challenge
    alt Hợp lệ
        A-->>C: Access token, ID token nếu OIDC, refresh token nếu cấp
        C->>C: Xác thực ID token nếu dùng OIDC; xóa giao dịch tạm
        C->>R: Authorization: Bearer access-token
        R-->>C: Kết quả sau xác thực và phân quyền
    else Không hợp lệ
        A-->>C: Lỗi token endpoint, không cấp token
        C->>C: Kết thúc giao dịch; đăng nhập lại khi cần
    end
```

Client trong sơ đồ có thể là SPA, mobile hoặc backend. Với backend dùng session, browser chỉ giữ cookie phiên; backend đổi code và giữ token như [flow OIDC](DocumentOIDC.md#3-logic-đăng-nhập-oidc).

## 4. Ví dụ HTTP từng bước

### 4.1 Tạo dữ liệu giao dịch

Ví dụ Java dùng JDK 17 trở lên:

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

public class PkceExample {
    public static void main(String[] args) throws Exception {
        SecureRandom random = new SecureRandom();
        byte[] bytes = new byte[32];
        random.nextBytes(bytes);
        String verifier = Base64.getUrlEncoder().withoutPadding()
                .encodeToString(bytes);
        String challenge = Base64.getUrlEncoder().withoutPadding()
                .encodeToString(MessageDigest.getInstance("SHA-256")
                        .digest(verifier.getBytes(StandardCharsets.US_ASCII)));
        // Gắn verifier với giao dịch đăng nhập; gửi challenge tới authorization endpoint.
        // Không ghi verifier vào log hoặc đưa nó vào authorization URL.
    }
}
```

Sinh `state` và `nonce` độc lập, lưu cùng giao dịch và thời hạn ngắn. Cho phép nhiều giao dịch theo từng state để tránh ghi đè khi mở nhiều tab; xóa dữ liệu khi hoàn tất hoặc hết hạn.

### 4.2 Chuyển browser tới authorization endpoint

Ví dụ public client OIDC. `CHALLENGE`, `STATE`, `NONCE` là placeholder; thay và URL-encode giá trị trước khi gửi. Đường dẫn endpoint phụ thuộc provider.

```http
GET /authorize?response_type=code&client_id=learning-spa&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback&scope=openid%20profile%20orders.read&state=STATE&nonce=NONCE&code_challenge=CHALLENGE&code_challenge_method=S256 HTTP/1.1
Host: identity.example.com
```

### 4.3 Callback

```http
HTTP/1.1 302 Found
Location: https://app.example.com/callback?code=AUTHORIZATION_CODE&state=STATE
```

Client kiểm tra state trước khi đổi code. Nếu callback chứa `error=access_denied`, kết thúc giao dịch; không tiếp tục như đã đăng nhập.

### 4.4 Đổi code tại token endpoint

Public client không gửi secret:

```http
POST /token HTTP/1.1
Host: identity.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&client_id=learning-spa&code=AUTHORIZATION_CODE&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback&code_verifier=VERIFIER
```

Confidential client còn xác thực theo đăng ký, chẳng hạn `client_secret_basic`. Redirect URI phải nhất quán với yêu cầu ban đầu. Server chỉ chấp nhận code còn hạn, chưa sử dụng, đúng client và verifier. Xem [token request của OAuth2](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.1.3).

```json
{
  "access_token": "example-access-token",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "openid profile orders.read",
  "id_token": "example-id-token"
}
```

Thời hạn là minh họa. Refresh token không được bảo đảm sẽ cấp. Với OIDC, kiểm tra chữ ký, issuer, audience, thời hạn và nonce đã gửi trước khi chấp nhận đăng nhập. Xem [OIDC ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation).

### 4.5 Gọi API

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer example-access-token
```

Resource Server nhận access token, không nhận code/verifier/ID token. PKCE không tham gia lại ở mỗi request API. Xem [Resource Server](DocumentResourceServer.md).

## 5. Áp dụng trong Spring Security

| Mô hình | Client thực hiện PKCE | Vai trò Spring Boot |
| --- | --- | --- |
| SPA/mobile gọi API trực tiếp | Thư viện OAuth2/OIDC của SPA/mobile | Resource Server. |
| Web backend / BFF giữ token | OAuth2 Client ở backend | OIDC Login, session và gọi API nếu cần. |

Với Servlet OAuth2 Client, Spring cung cấp `DefaultOAuth2AuthorizationRequestResolver` và cơ chế quản lý authorization request. Public client đăng ký `client-authentication-method: none`; PKCE được áp dụng theo điều kiện của framework. Không đổi confidential client thành public chỉ để bật PKCE.

Ví dụ phần registration cho **Servlet client đã được provider đăng ký là public**, không phải cấu hình frontend SPA:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          learning:
            provider: company
            client-id: learning-public-client
            client-authentication-method: none
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: openid,profile
        provider:
          company:
            issuer-uri: ${OIDC_ISSUER_URI}
```

Cần dependency OAuth2 Client và `oauth2Login()` như [cấu hình OIDC](DocumentOIDC.md#32-securityfilterchain-cho-ứng-dụng-web). Provider phải hỗ trợ public client và PKCE. Với confidential client, đối chiếu cách yêu cầu PKCE trong đúng phiên bản; không mặc định mọi cấu hình tự bật. Xem [Spring Security Authorization Grant Support](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorization-grants.html).

## 6. Giới hạn và kiểm tra triển khai

PKCE không bảo vệ token đã bị đánh cắp sau khi cấp, không khắc phục XSS hay thay HTTPS. Dùng S256; không hạ xuống `plain`. Public client phải dùng PKCE theo hướng dẫn hiện hành; confidential client cũng được khuyến nghị sử dụng. Xem [RFC 9700, mục 2.1.1](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1.1).

| Tình huống | Kết quả cần kiểm chứng |
| --- | --- |
| Flow hợp lệ | Nhận token; chỉ API đủ quyền mới được gọi. |
| Callback sai hoặc thiếu state so với giao dịch đã tạo | Client từ chối callback. |
| Verifier sai hoặc thiếu khi code đã gắn challenge | Token endpoint từ chối. |
| Dùng lại code hoặc code hết hạn | Token endpoint từ chối, thường `invalid_grant`. |
| Client hoặc redirect URI không khớp | Server từ chối theo lỗi giao thức tương ứng. |
| ID token sai nonce đã gửi | OIDC client từ chối đăng nhập. |
| Token hợp lệ nhưng thiếu scope API | API từ chối quyền, thường 403. |

Đây là tiêu chí kiểm tra, chưa phải kết quả chạy tích hợp của repo. Khi quan sát request để xác minh S256/verifier, che dữ liệu nhạy cảm và không lưu code, verifier hay token vào log.

[Quay về mục lục Spring Security](README.md)
