# Spring Authorization Server

Đọc sau [Authorization Code + PKCE](DocumentAuthorizationCodePKCE.md). Tài liệu giải thích cách xây dựng bên cấp token, kèm cấu hình và bài thực hành. Repository hiện chỉ chứa tài liệu; các đoạn cấu hình dưới đây chưa tạo thành một module Java và chưa được chạy kiểm thử.

## 1. Hiểu đúng vai trò

Ví dụ một hệ thống có Angular, backend quản lý đơn hàng và một dịch vụ đăng nhập tập trung:

| Thành phần | Trách nhiệm trong ví dụ |
| --- | --- |
| Angular hoặc backend BFF | OAuth2 Client: khởi tạo giao dịch, nhận kết quả đăng nhập. |
| Dịch vụ đăng nhập | Authorization Server: xử lý giao thức, cấp code và token. |
| API đơn hàng | Resource Server: kiểm tra access token và quyền truy cập dữ liệu. |

Spring Authorization Server cung cấp nền tảng để lập trình bên cấp token. Bạn vẫn cần xây dựng các phần của sản phẩm như quản lý tài khoản, giao diện quản trị và quy trình khôi phục tài khoản. Có thể dùng nó khi cần tùy biến sâu bằng Java/Spring; khi đánh giá một sản phẩm IAM hoàn chỉnh, hãy đối chiếu riêng các nhu cầu vận hành đó.

OAuth2 phục vụ ủy quyền; OIDC bổ sung lớp định danh để đăng nhập. Chỉ phát một chuỗi JWT từ controller `/login` chưa đủ để triển khai các giao thức này.

## 2. Chọn đúng nhánh phiên bản

Spring đã đưa Authorization Server vào Spring Security từ nhánh 7.0; artifact vẫn là `spring-security-oauth2-authorization-server`. Xem [thông báo chính thức](https://spring.io/blog/2025/09/11/spring-authorization-server-moving-to-spring-security-7-0/).

Ví dụ trong file này lấy **Spring Boot 3.5.x / Spring Security 6.5.x / Authorization Server 1.5.x, Java 17+** làm mốc học. Dùng dependency management của Boot để chọn bản tương thích, không ghép thư viện Security 7 vào Boot 3.5. Đây là mốc cho ví dụ, không phải khẳng định phiên bản mới nhất.

Với Boot 4 / Security 7, đọc [tài liệu Authorization Server trong Spring Security](https://docs.spring.io/spring-security/reference/servlet/oauth2/authorization-server/getting-started.html) trước khi chuyển cấu hình.

## 3. Flow cấp token và gọi API

```mermaid
sequenceDiagram
    actor U as Người dùng / Browser
    participant C as Client / BFF
    participant A as Authorization Server
    participant R as API / Resource Server
    C-->>U: Redirect kèm state và PKCE challenge
    U->>A: GET /oauth2/authorize
    A-->>U: Yêu cầu đăng nhập nếu chưa có phiên
    U->>A: Đăng nhập và đồng ý quyền nếu cần
    A-->>U: Redirect callback với code và state
    U->>C: Callback
    C->>C: Kiểm tra state
    C->>A: POST /oauth2/token với code và verifier
    Note over C,A: BFF còn xác thực bằng thông tin client
    A-->>C: Access token, ID token nếu OIDC, refresh token nếu được cấp
    C->>C: Xác thực ID token nếu dùng OIDC
    C->>R: Bearer access token
    R->>R: Kiểm tra chữ ký, issuer, thời hạn, audience và quyền
    R-->>C: Kết quả API
```

Đăng nhập tại Authorization Server thường dùng session. API có thể kiểm tra JWT theo từng request. Vì vậy, dùng JWT cho API không có nghĩa mọi thành phần đều phải stateless.

Trong ví dụ đơn hàng: `openid` yêu cầu đăng nhập OIDC, còn `orders.read` thể hiện quyền đọc API. ID token dành cho client xác thực kết quả đăng nhập; access token dành cho API. Xem [flow PKCE chi tiết](DocumentAuthorizationCodePKCE.md#3-flow-đầy-đủ).

## 4. Các endpoint cần nhận biết

Các đường dẫn mặc định có thể được thay đổi qua cấu hình:

| Endpoint | Mục đích |
| --- | --- |
| `/oauth2/authorize` | Bắt đầu ủy quyền qua browser. |
| `/oauth2/token` | Đổi grant lấy token. |
| `/oauth2/jwks` | Công bố khóa công khai để kiểm tra chữ ký. |
| `/.well-known/oauth-authorization-server` | Metadata OAuth2. |
| `/.well-known/openid-configuration` | Discovery OIDC khi bật OIDC. |
| `/userinfo` | Thông tin người dùng theo OIDC và quyền đã cấp. |
| `/oauth2/introspect` | Kiểm tra trạng thái token bằng client được phép. |
| `/oauth2/revoke` | Yêu cầu thu hồi token. |
| `/connect/logout` | Đăng xuất OIDC khi bật tính năng tương ứng. |

Đây là endpoint giao thức do filter xử lý; không cần tự viết controller cho từng đường dẫn. Các điểm mở rộng gồm converter đọc request, authentication provider xử lý, và handler trả kết quả. Xem [Protocol Endpoints](https://docs.spring.io/spring-authorization-server/reference/protocol-endpoints.html).

## 5. Những thành phần cần phân biệt

| Thành phần | Dữ liệu / trách nhiệm |
| --- | --- |
| `RegisteredClientRepository` | Client được đăng ký, grant, redirect URI, scope và secret đã mã hóa. |
| `UserDetailsService` hoặc cơ chế đăng nhập khác | Người dùng của hệ thống. |
| `OAuth2AuthorizationService` | Trạng thái authorization, code và token liên quan. |
| `OAuth2AuthorizationConsentService` | Những quyền người dùng đã đồng ý cho client. |
| `JWKSource<SecurityContext>` | Khóa phục vụ ký JWT và công bố khóa công khai. |
| `AuthorizationServerSettings` | Issuer và địa chỉ endpoint. |
| `ClientSettings` / `TokenSettings` | PKCE, consent, thời hạn và chính sách token. |

`RegisteredClient` ở server và `ClientRegistration` ở ứng dụng client là hai model khác nhau. Mật khẩu người dùng cũng khác client secret. Khi chuyển từ demo sang lưu trữ lâu dài, xem các implementation JDBC cho client, authorization và consent trong [Core Model / Components](https://docs.spring.io/spring-authorization-server/reference/core-model-components.html).

## 6. Cấu hình minh họa với Spring Boot

### 6.1 Dependency

Trong một project Boot 3.5.x có dependency management, thêm starter sau; dùng Spring Web cho ứng dụng servlet:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 6.2 Cấu hình bằng properties cho bài học

```yaml
server:
  port: 9000

spring:
  security:
    user:
      name: learner
      password: "${DEMO_USER_PASSWORD}"
    oauth2:
      authorizationserver:
        issuer: http://localhost:9000
        client:
          learning-bff:
            registration:
              client-id: learning-bff
              client-secret: "${DEMO_CLIENT_SECRET_HASH}"
              client-authentication-methods:
                - client_secret_basic
              authorization-grant-types:
                - authorization_code
                - refresh_token
              redirect-uris:
                - http://127.0.0.1:8080/login/oauth2/code/learning-bff
              scopes:
                - openid
                - profile
                - orders.read
            require-authorization-consent: true
            require-proof-key: true
```

`DEMO_CLIENT_SECRET_HASH` chứa giá trị theo định dạng `PasswordEncoder`, chẳng hạn `{bcrypt}` kèm hash hợp lệ. Client giữ secret gốc để xác thực, không gửi hash thay secret. `DEMO_USER_PASSWORD` phục vụ tài khoản demo; môi trường thực cần cơ chế lưu người dùng riêng.

Cấu hình này dựa vào auto-configuration của starter. Callback phải có một ứng dụng OAuth2 Client đang chạy ở cổng 8080 với registration ID `learning-bff`; chỉ chạy Authorization Server chưa tạo ra client đó. Xem [Getting Started](https://docs.spring.io/spring-authorization-server/reference/getting-started.html).

### 6.3 Khi tự khai báo filter chain

Đoạn sau minh họa phần bảo mật trong một lớp `@Configuration`, dùng API của nhánh 1.5.x. Khi chọn cách này, cần cung cấp đủ các bean như client repository, cơ chế đăng nhập, khóa ký, `JwtDecoder` cho OIDC và settings theo hướng dẫn chính thức; không coi đây là cấu hình độc lập có thể chạy ngay.

```java
@Bean
@Order(1)
SecurityFilterChain protocolChain(HttpSecurity http) throws Exception {
    var server = OAuth2AuthorizationServerConfigurer.authorizationServer();
    http.securityMatcher(server.getEndpointsMatcher())
        .with(server, config -> config.oidc(Customizer.withDefaults()))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .exceptionHandling(errors -> errors.defaultAuthenticationEntryPointFor(
            new LoginUrlAuthenticationEntryPoint("/login"),
            new MediaTypeRequestMatcher(MediaType.TEXT_HTML)));
    return http.build();
}

@Bean
@Order(2)
SecurityFilterChain loginChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

Các kiểu Spring cần import:

```text
org.springframework.context.annotation.Bean
org.springframework.core.annotation.Order
org.springframework.http.MediaType
org.springframework.security.config.Customizer
org.springframework.security.config.annotation.web.builders.HttpSecurity
org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer
org.springframework.security.web.SecurityFilterChain
org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint
org.springframework.security.web.util.matcher.MediaTypeRequestMatcher
```

Chain đầu khớp endpoint giao thức; chain sau xử lý đăng nhập và các request còn lại. Một request đi vào chain khớp đầu tiên theo thứ tự. Không đặt chain bắt mọi request trước chain giao thức. Với cấu hình thủ công, OIDC cần được bật tường minh. Xem [Configuration Model](https://docs.spring.io/spring-authorization-server/reference/configuration-model.html).

## 7. Chọn client theo ứng dụng

| Ứng dụng | Cách thiết kế |
| --- | --- |
| Backend web / BFF | Confidential client, giữ secret phía server; Authorization Code + PKCE. |
| Angular gọi API trực tiếp | Public client, authentication method `none`, bắt buộc PKCE; không nhúng secret vào bundle. |
| Job đồng bộ giữa dịch vụ | Client Credentials, không có bước đăng nhập người dùng. |

Với public client, đăng ký `authorization_code`, redirect URI chính xác và `requireProofKey(true)`. Nhánh SAS 1.5.x không cấp refresh token cho public client theo [hướng dẫn SPA chính thức](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-pkce.html); không suy ra rằng thêm grant `refresh_token` sẽ giải quyết được. BFF là một hướng phù hợp khi cần giữ token phía server.

SPA khác origin còn cần cấu hình CORS cho các request thực sự gọi bằng JavaScript. Redirect browser tới authorization endpoint và CORS là hai cơ chế khác nhau.

## 8. Thử flow và đọc kết quả

### 8.1 Kiểm tra discovery

Sau khi tạo project demo và chạy ở cổng 9000, dùng PowerShell:

```powershell
$metadata = Invoke-RestMethod 'http://localhost:9000/.well-known/openid-configuration'
$metadata | Select-Object issuer, authorization_endpoint, token_endpoint, jwks_uri
Invoke-RestMethod $metadata.jwks_uri
```

Mong đợi issuer đúng `http://localhost:9000` và JWK Set chứa khóa công khai. Không xuất private key qua HTTP.

### 8.2 Thử đăng nhập bằng OAuth2 Client hoặc Postman

1. Cấu hình Authorization Code + PKCE với authorization URL và token URL từ discovery.
2. Dùng client đã đăng ký; callback của công cụ phải khớp redirect URI được server cho phép. Nếu dùng Postman, đăng ký callback mà Postman hiển thị thay vì dùng nguyên callback của BFF.
3. Với confidential client, chọn gửi client credentials bằng HTTP Basic và nhập secret gốc.
4. Yêu cầu `openid profile orders.read`, dùng challenge method `S256`, rồi đăng nhập bằng browser và chấp thuận nếu được hỏi.
5. Đối chiếu callback, kết quả token và gọi API bằng access token.

Token request sau callback có dạng sau; các giá trị viết hoa là placeholder, không phải token thật:

```http
POST /oauth2/token HTTP/1.1
Host: localhost:9000
Authorization: Basic BASE64_CLIENT_ID_COLON_SECRET
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=CODE_MOI_NHAN&redirect_uri=REDIRECT_URI_DA_URL_ENCODE&code_verifier=VERIFIER_CUA_GIAO_DICH
```

Code chỉ dùng một lần. Khi thử lại, tạo giao dịch mới; không tái dùng code cũ. Ví dụ đổi refresh token cho confidential client:

```http
POST /oauth2/token HTTP/1.1
Host: localhost:9000
Authorization: Basic BASE64_CLIENT_ID_COLON_SECRET
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=REFRESH_TOKEN_DA_URL_ENCODE
```

Kiểm tra response để biết có refresh token mới hay không, và cập nhật nơi lưu theo chính sách server. Client Credentials dùng grant riêng cho danh tính dịch vụ; scope và grant đó phải được đăng ký riêng cho client tương ứng.

### 8.3 Nối API Resource Server

Trong **ứng dụng API riêng**, thêm starter `spring-boot-starter-oauth2-resource-server`, cấu hình `issuer-uri: http://localhost:9000` và bật `oauth2ResourceServer(...jwt...)` trong chain của API. Quyền `orders.read` thường được ánh xạ thành authority `SCOPE_orders.read`.

API cần kiểm tra audience dự kiến bên cạnh issuer, chữ ký và thời hạn. Đồng thời Authorization Server phải cấp claim `aud` phù hợp; không giả định tên scope tự trở thành audience. Xem [Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html) và [tài liệu Resource Server trong repo](DocumentResourceServer.md).

## 9. Claims, role và dữ liệu nghiệp vụ

Scope mô tả quyền client được cấp; role mô tả vai trò theo mô hình của ứng dụng. Quy ước mapping phải thống nhất giữa bên phát và bên đọc token.

Khi cần thêm `roles`, `tenant_id` hoặc audience, có thể dùng `OAuth2TokenCustomizer<JwtEncodingContext>` và kiểm tra loại token trước khi thêm claim. Nguồn dữ liệu phải đáng tin cậy; không lấy role trực tiếp từ tham số client gửi lên. Với Client Credentials, principal là client, không phải một người dùng đang đăng nhập.

Resource Server cần converter riêng nếu muốn đổi claim `roles` thành authority. Thêm claim vào JWT chưa tự thay đổi `hasRole(...)`. Xem [hướng dẫn custom claims](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-custom-claims-authorities.html).

## 10. Khi đưa ví dụ vào hệ thống thực

Các quyết định cần chốt cho bài toán của bạn:

- **Dữ liệu:** lưu client, authorization và consent bền vững; quản lý schema bằng migration, không giữ mọi thứ trong bộ nhớ.
- **Khóa:** giữ khóa ký ổn định giữa các lần restart và các instance; dùng `kid` và có giai đoạn chồng lấn khóa khi rotation để token cũ còn xác minh được.
- **Issuer:** dùng địa chỉ HTTPS công khai ổn định; kiểm tra reverse proxy để discovery không trả host nội bộ.
- **Đăng nhập:** bảo vệ form bằng CSRF, quản lý session và cookie phù hợp. Không áp dụng cấu hình stateless của API lên toàn bộ server đăng nhập.
- **Token:** quyết định thời hạn, rotation, cách lưu token, thu hồi và chính sách phiên theo yêu cầu nghiệp vụ.
- **Quyền:** giới hạn scope theo client, kiểm tra quyền trên dữ liệu đơn hàng ngay tại API.

Logout, revoke refresh token và vô hiệu hóa access token là những hành động khác nhau. Một JWT đã phát có thể vẫn được API xác thực offline đến khi hết hạn; nếu yêu cầu khóa ngay, cần thiết kế thêm cơ chế kiểm tra trạng thái hoặc thu hồi phù hợp. Xem [vòng đời token](DocumentJWT.md).

## 11. Lỗi thường gặp và thứ tự kiểm tra

| Hiện tượng | Kiểm tra trước |
| --- | --- |
| `invalid_client` | Client ID, secret gốc, hash lưu ở server, authentication method. |
| `invalid_grant` | Code hết hạn/đã dùng, verifier sai, redirect URI lệch hoặc refresh token không còn hợp lệ. |
| `invalid_scope` | Scope có được đăng ký và được phép cho giao dịch đó không. |
| Không redirect về client | Redirect URI, callback đang chạy, request ban đầu và lỗi tại authorization endpoint. |
| Lặp lại trang login | Chain matcher/order, cookie phiên, host và proxy. |
| API trả 401 | Token đúng loại, issuer, audience, thời hạn, `kid` và khóa công khai. |
| API trả 403 | Scope/authority hoặc điều kiện phân quyền dữ liệu. |
| Restart xong không dùng được token | Có mất authorization trong bộ nhớ hoặc thay khóa ký không. |

Ghi lại endpoint, grant, mã lỗi và correlation ID; tránh ghi password, secret hoặc nguyên token vào log. Khi debug, đi theo từng chặng: đăng nhập → callback → đổi code → kiểm tra token → phân quyền API.

## 12. Bài tập tự kiểm tra

1. Đăng ký BFF, đăng nhập OIDC rồi gọi API yêu cầu `orders.read`.
2. Đổi verifier để quan sát lần đổi code bị từ chối.
3. Đăng ký một client chỉ có scope khác và thử gọi API đơn hàng.
4. Phát token, restart server demo rồi đối chiếu ảnh hưởng của dữ liệu trong bộ nhớ và khóa ký.
5. Thu hồi refresh token, thử refresh lại, rồi quan sát riêng access token đang còn hạn.

Sau mỗi bài, giải thích thành phần nào đưa ra quyết định và dữ liệu nào được kiểm tra. Đây là cách phân biệt rõ trách nhiệm của Authorization Server, client và API.

[Quay về mục lục Spring Security](README.md)
