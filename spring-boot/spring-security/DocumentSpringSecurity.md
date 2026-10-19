# Spring Security

Tài liệu giới thiệu các khái niệm xác thực, phân quyền và luồng xử lý request của Spring Security trong ứng dụng Servlet như Spring Boot với Spring MVC. Nội dung tập trung vào đăng nhập bằng username/password, xác thực JWT và đăng nhập qua OAuth2/OpenID Connect.

Các ví dụ là những đoạn minh họa riêng lẻ, không phải một ứng dụng hoàn chỉnh. Tên filter và hành vi cụ thể phụ thuộc vào phiên bản Spring Security và cấu hình đang sử dụng.

## 1. Spring Security là gì

Spring Security là framework bảo mật trong hệ sinh thái Spring, cung cấp cơ chế **xác thực** (authentication), **phân quyền** (authorization) và các tính năng bảo vệ ứng dụng web.

### Khái niệm cơ bản

- **Authentication — Xác thực:** xác định người dùng hoặc client là ai, chẳng hạn qua username/password hoặc access token.
- **Authorization — Phân quyền:** xác định người dùng hoặc client được phép làm gì, chẳng hạn chỉ người có quyền quản trị mới được truy cập `/admin/**`.

Các cơ chế và khả năng tích hợp thường gặp:

- Form Login.
- HTTP Basic Authentication.
- OAuth2 và OpenID Connect (OIDC).
- OAuth2 Resource Server với JWT hoặc opaque token.
- LDAP.
- Single Sign-On (SSO), thông qua những giao thức và hệ thống định danh phù hợp.

JWT là một định dạng token; OAuth2 là giao thức ủy quyền; OIDC bổ sung lớp xác thực danh tính trên OAuth2. Chúng có thể kết hợp với nhau, không phải ba khái niệm tương đương.

## 2. Kiến trúc Spring Security

### Các thành phần chính

| Thành phần | Vai trò |
| --- | --- |
| `DelegatingFilterProxy` | Kết nối Servlet filter với bean do Spring quản lý. |
| `FilterChainProxy` | Chọn `SecurityFilterChain` phù hợp với request. |
| `SecurityFilterChain` | Xác định các filter bảo mật áp dụng cho request phù hợp. |
| `Authentication` | Biểu diễn yêu cầu xác thực hoặc kết quả xác thực, gồm principal, credentials và authorities tùy trạng thái. |
| `AuthenticationManager` | API thực hiện xác thực thông qua `authenticate(...)`. |
| `ProviderManager` | Implementation thường dùng của `AuthenticationManager`, điều phối các provider hỗ trợ kiểu `Authentication` đầu vào. |
| `AuthenticationProvider` | Thực hiện logic xác thực cho một kiểu token nhất định. |
| `UserDetailsService` | Tải `UserDetails` từ cơ sở dữ liệu hoặc nguồn lưu trữ khác, thường dùng trong xác thực username/password. |
| `PasswordEncoder` | Băm mật khẩu và kiểm tra mật khẩu nhập vào với giá trị đã lưu. |
| `GrantedAuthority` | Biểu diễn một quyền, ví dụ `ROLE_ADMIN` hoặc `SCOPE_read`. |
| `SecurityContext` | Chứa `Authentication` của người dùng hiện tại. |
| `SecurityContextHolder` | Cho phép truy cập `SecurityContext`; mặc định sử dụng `ThreadLocal`. |
| `SecurityContextRepository` | Lưu và tải context giữa các request khi cấu hình yêu cầu. |

Nguồn đối chiếu: [kiến trúc Servlet](https://docs.spring.io/spring-security/reference/servlet/architecture.html) và [kiến trúc xác thực](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html).

### Các tính năng thường dùng

- **CSRF protection:** bảo vệ chống Cross-Site Request Forgery. Với cấu hình mặc định có CSRF, request thay đổi trạng thái như `POST /login` cần CSRF token hợp lệ.
- **CORS:** kiểm soát việc trình duyệt cho phép một origin khác gọi ứng dụng. CORS không thay thế xác thực hoặc phân quyền.
- **Remember-Me:** dùng cookie/token để nhận diện lại người dùng sau khi phiên đăng nhập kết thúc, tùy cấu hình.
- **PasswordEncoder:** dùng các hàm băm mật khẩu như BCrypt hoặc Argon2. Đây là biến đổi một chiều, không phải mã hóa có thể giải mã để lấy lại mật khẩu gốc. Xem [Password Storage](https://docs.spring.io/spring-security/reference/7.0/features/authentication/password-storage.html).
- **Method Security:** kiểm tra quyền khi gọi method của bean do Spring quản lý.

Ví dụ Method Security:

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) {
    // Xóa người dùng.
}
```

Cần bật `@EnableMethodSecurity` trong lớp cấu hình để sử dụng `@PreAuthorize`. Theo quy ước mặc định, `hasRole("ADMIN")` kiểm tra authority `ROLE_ADMIN`. Xem [Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html).

## 3. Tích hợp JWT

Một REST API sử dụng JWT thường có luồng như sau:

1. Người dùng đăng nhập tại hệ thống cấp token.
2. Hệ thống cấp access token, có thể ở định dạng JWT.
3. Client gửi token trong những request tiếp theo:

   ```http
   Authorization: Bearer <token>
   ```

4. API kiểm tra token và thiết lập `Authentication` cho request.
5. API kiểm tra quyền truy cập trước khi xử lý nghiệp vụ.

Spring Security hỗ trợ xác thực JWT bằng OAuth2 Resource Server. Không bắt buộc phải tự viết filter JWT. Luồng tích hợp sẵn thường là:

```text
BearerTokenAuthenticationFilter
    → AuthenticationManager
    → JwtAuthenticationProvider
    → JwtDecoder
    → JwtAuthenticationToken
```

`JwtDecoder` giải mã và kiểm tra token, bao gồm chữ ký và các điều kiện xác thực đã cấu hình như thời hạn, issuer hoặc audience. Chỉ đọc payload không đủ để xác thực JWT. Kết quả thành công được đưa vào `SecurityContext` để phục vụ phân quyền. Xem [OAuth2 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html).

Việc API kiểm tra JWT và việc hệ thống cấp JWT là hai vai trò riêng. Cấu hình Resource Server không tự tạo endpoint đăng nhập hoặc cấp token.

## 4. OAuth2 và OpenID Connect

Spring Security hỗ trợ đăng nhập qua các nhà cung cấp như Google, GitHub hoặc Facebook bằng OAuth2 Login; với nhà cung cấp hỗ trợ OIDC, ứng dụng có thể dùng OIDC để nhận thông tin danh tính.

Cần có dependency phù hợp, cấu hình client registration như client ID, client secret, redirect URI và bật `oauth2Login(...)`. Các thông tin đăng ký client có thể được khai báo trong `application.yml`.

Luồng Authorization Code thường gồm:

1. Trình duyệt gọi endpoint khởi động đăng nhập của ứng dụng, ví dụ `/oauth2/authorization/google`.
2. Ứng dụng chuyển hướng trình duyệt sang nhà cung cấp.
3. Người dùng đăng nhập và chấp thuận quyền truy cập nếu được yêu cầu.
4. Nhà cung cấp chuyển hướng trình duyệt về callback của ứng dụng, mặc định dạng `/login/oauth2/code/{registrationId}`, kèm authorization code.
5. Ứng dụng đổi code lấy token và tải thông tin người dùng hoặc xử lý thông tin danh tính OIDC.
6. Spring Security tạo `Authentication` tương ứng.

`OAuth2AuthorizationRequestRedirectFilter` xử lý bước bắt đầu chuyển hướng; `OAuth2LoginAuthenticationFilter` xử lý callback đăng nhập. Xem [OAuth2](https://docs.spring.io/spring-security/reference/servlet/oauth2/) và [OAuth2 Login](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/index.html).

## 5. Luồng xử lý request

### Bước 1 — Client gửi request

Client có thể là trình duyệt, ứng dụng di động, Postman hoặc một API client khác. Trong OAuth2 Login, nhà cung cấp thường chuyển hướng trình duyệt về ứng dụng qua callback.

Các request phổ biến:

**Đăng nhập bằng form:**

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=john&password=example-password&_csrf=<csrf-token>
```

Tên tham số CSRF mặc định là `_csrf`; giá trị phải lấy từ ứng dụng. Nếu dùng CSRF token qua header, cách gửi phụ thuộc cấu hình. Request thành công có thể tạo phiên đăng nhập; form login mặc định không tự trả JWT.

**Gọi API bằng JWT:**

```http
GET /api/user/profile HTTP/1.1
Authorization: Bearer <jwt-access-token>
```

**Gọi API bằng HTTP Basic:**

```http
GET /api/data HTTP/1.1
Authorization: Basic <base64-username-colon-password>
```

HTTP Basic mã hóa biểu diễn `username:password` bằng Base64, không mã hóa để giữ bí mật; cần HTTPS để bảo vệ thông tin khi truyền.

**Bắt đầu OAuth2 Login:**

```http
GET /oauth2/authorization/google HTTP/1.1
```

**Truy cập endpoint công khai:**

```http
GET /api/public/news HTTP/1.1
```

Endpoint công khai cần được cho phép trong cấu hình phân quyền, chẳng hạn bằng `permitAll()`.

Request đi vào Servlet container như Tomcat hoặc Jetty, sau đó qua Servlet filter chain trước khi đến controller.

### Bước 2 — Security Filter Chain xử lý request

Spring Security tích hợp vào Servlet filter chain qua `DelegatingFilterProxy` và `FilterChainProxy`. Nếu có nhiều `SecurityFilterChain`, `FilterChainProxy` chọn **chain đầu tiên khớp request** theo thứ tự cấu hình; không chạy lần lượt tất cả các security chain.

Các filter trong chain được chọn thực hiện những nhiệm vụ riêng: tải context, xử lý CSRF, xác thực, xử lý lỗi hoặc phân quyền. Không phải filter nào cũng tạo yêu cầu xác thực.

Một filter có thể:

- Gọi `chain.doFilter(request, response)` để chuyển tiếp request.
- Xử lý request rồi chuyển tiếp, tùy cơ chế.
- Kết thúc xử lý bằng response hoặc redirect, chẳng hạn khi đăng nhập thất bại hoặc khi cần chuyển hướng đến trang đăng nhập.

Một số filter tiêu biểu:

| Filter | Nhiệm vụ |
| --- | --- |
| `UsernamePasswordAuthenticationFilter` | Xử lý request đăng nhập bằng form. |
| `BasicAuthenticationFilter` | Xử lý thông tin đăng nhập HTTP Basic. |
| `BearerTokenAuthenticationFilter` | Xử lý Bearer token khi dùng OAuth2 Resource Server. |
| `OAuth2LoginAuthenticationFilter` | Xử lý callback OAuth2/OIDC Login. |
| `SecurityContextHolderFilter` | Tải context từ repository trong cấu hình dùng cơ chế lưu tường minh. |
| `SecurityContextPersistenceFilter` | Tải và lưu context trong cơ chế cũ; không phải mặc định của Spring Security 6. |
| `ExceptionTranslationFilter` | Chuyển lỗi xác thực hoặc từ chối truy cập ở các bước phía sau thành cách phản hồi phù hợp. |
| `AuthorizationFilter` | Kiểm tra phân quyền request khi sử dụng `authorizeHttpRequests(...)`. |

`FilterSecurityInterceptor` thường xuất hiện trong cấu hình phân quyền cũ. Với `authorizeHttpRequests(...)`, thành phần thường gặp là `AuthorizationFilter`. Xem [kiến trúc Servlet](https://docs.spring.io/spring-security/reference/servlet/architecture.html) và [phân quyền HTTP request](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html).

### Bước 3 — AuthenticationManager và AuthenticationProvider

Filter xác thực tạo một `Authentication` đầu vào và gọi manager. Ví dụ minh họa với username/password:

```java
UsernamePasswordAuthenticationToken authRequest =
        new UsernamePasswordAuthenticationToken(username, password);

Authentication authResult =
        authenticationManager.authenticate(authRequest);
```

`authRequest` ở trạng thái chưa xác thực. Lời gọi `authenticate(...)` **thực hiện quá trình xác thực**: nếu thành công, trả về kết quả đã xác thực; nếu thất bại, có thể ném `AuthenticationException`.

`AuthenticationManager` là interface. Với implementation thường dùng là `ProviderManager`, việc xác thực được giao cho các `AuthenticationProvider` phù hợp:

1. Kiểm tra provider nào hỗ trợ kiểu `Authentication` bằng `supports(...)`.
2. Gọi `provider.authenticate(...)` để thực hiện xác thực.
3. Trả kết quả thành công hoặc xử lý thất bại theo quy tắc của manager.

`supports(...)` chỉ cho biết provider hỗ trợ kiểu token, không xác nhận thông tin đăng nhập hợp lệ. Provider có thể trả `null` để provider khác tiếp tục xử lý; không nên hiểu rằng manager luôn dừng tại provider đầu tiên có `supports(...)` trả `true`.

### Bước 4 — Những cơ chế xác thực thường gặp

#### Đăng nhập bằng username và password

- **Filter:** `UsernamePasswordAuthenticationFilter`.
- **Provider:** `DaoAuthenticationProvider`.
- **Nguồn thông tin người dùng:** `UserDetailsService`.
- **Kiểm tra mật khẩu:** `PasswordEncoder`.

Provider tải người dùng, kiểm tra trạng thái tài khoản và so khớp mật khẩu. Nếu hợp lệ, trả về `Authentication` đã xác thực với các authorities tương ứng.

```text
UsernamePasswordAuthenticationFilter
    → AuthenticationManager / ProviderManager
    → DaoAuthenticationProvider
        → UserDetailsService
        → PasswordEncoder
    → Authentication đã xác thực
```

#### Kiểm tra JWT

- **Filter tích hợp sẵn:** `BearerTokenAuthenticationFilter`.
- **Provider:** `JwtAuthenticationProvider`.
- **Kiểm tra token:** `JwtDecoder` và các validator đã cấu hình.
- **Kết quả thường gặp:** `JwtAuthenticationToken`.

Filter đọc Bearer token, manager gọi provider và provider kiểm tra JWT. Một `JwtAuthenticationFilter` tự viết là lựa chọn triển khai riêng, không phải tên filter JWT chuẩn của Spring Security 6.

#### Đăng nhập bằng OAuth2 hoặc OIDC

- **Filter xử lý callback:** `OAuth2LoginAuthenticationFilter`.
- **Provider tương ứng:** `OAuth2LoginAuthenticationProvider` hoặc `OidcAuthorizationCodeAuthenticationProvider`, tùy giao thức.
- **Xử lý:** đổi authorization code lấy token, tải hoặc xử lý thông tin người dùng, tạo kết quả xác thực.

Cả ba cơ chế đều tạo ra một `Authentication` đã xác thực khi thành công, nhưng dùng kiểu token và provider khác nhau.

### Bước 5 — Security Context

Sau khi xác thực thành công, cơ chế xác thực đặt kết quả vào `SecurityContext`. Các thành phần phía sau có thể truy cập thông tin người dùng qua `SecurityContextHolder`.

Minh họa việc thiết lập context:

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);
```

Đoạn code trên chỉ thiết lập context hiện tại; nếu tự xử lý đăng nhập và cần lưu giữa các request, phải lưu qua `SecurityContextRepository` phù hợp.

**Ứng dụng dùng session:** context có thể được lưu vào HTTP session và tải lại ở request tiếp theo. Các cơ chế đăng nhập tích hợp sẵn xử lý việc lưu theo cấu hình; không cần gửi lại username/password cho mỗi request.

**API JWT cấu hình stateless:** mỗi request gửi token và được xác thực riêng; thông tin xác thực không được duy trì bằng HTTP session. Dùng JWT không tự động làm ứng dụng stateless, vẫn cần cấu hình session phù hợp.

Trong Spring Security 6, `SecurityContextHolderFilter` là cơ chế mặc định để tải context và không tự lưu context. `SecurityContextPersistenceFilter` thuộc cơ chế tự động tải/lưu cũ. Sau khi xử lý request, context gắn với thread được dọn dẹp. Xem [Authentication Persistence and Session Management](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html).

Lấy thông tin xác thực hiện tại:

```java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```

Khi sử dụng, cần xét trường hợp không có authentication hoặc là người dùng anonymous. Không suy ra người dùng đã đăng nhập chỉ từ việc giá trị này khác `null`.

### Bước 6 — Authorization

Spring Security kiểm tra authorities của người dùng với quy tắc phân quyền. Có hai cấp thường dùng:

**Ở cấp HTTP request**, cấu hình trong `SecurityFilterChain`:

```java
http.authorizeHttpRequests(authorize -> authorize
        .requestMatchers("/api/public/**").permitAll()
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
);
```

**Ở cấp method**, dùng annotation như:

```java
@PreAuthorize("hasRole('ADMIN')")
```

Hai cấp có thể cùng áp dụng; request cần vượt qua các quy tắc tương ứng. Method Security cần bật `@EnableMethodSecurity` và lời gọi phải đi qua cơ chế proxy/interceptor của Spring. Xem [phân quyền HTTP request](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html) và [Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html).

Phản hồi thường gặp:

- **401 Unauthorized:** với API dùng Basic/Bearer, khi thiếu hoặc không có thông tin xác thực hợp lệ để truy cập tài nguyên cần bảo vệ.
- **403 Forbidden:** người dùng đã xác thực nhưng không đủ quyền; lỗi CSRF cũng có thể dẫn đến 403.
- **Redirect đến trang đăng nhập:** thường gặp với ứng dụng dùng form login khi người dùng chưa đăng nhập.

Phản hồi cụ thể phụ thuộc cơ chế xác thực, `AuthenticationEntryPoint`, `AccessDeniedHandler` và các handler đã cấu hình. Không phải mọi lỗi xác thực đều trả 403.

### Bước 7 — Controller hoặc resource xử lý nghiệp vụ

Sau khi vượt qua các filter và quy tắc phân quyền, request được chuyển đến controller hoặc resource để xử lý logic nghiệp vụ.

```java
@GetMapping("/admin/dashboard")
public String dashboard() {
    return "admin/dashboard";
}
```

Với quy tắc `/admin/**` ở trên, request tới endpoint này cần authority `ROLE_ADMIN`. Nếu controller là `@Controller`, chuỗi trả về thường là tên view; nếu là `@RestController`, chuỗi được trả trong response body.

## 6. Sơ đồ và bảng tóm tắt

```text
Client gửi request
    → Servlet container
    → DelegatingFilterProxy
    → FilterChainProxy chọn SecurityFilterChain
    → Các filter xử lý context, bảo vệ request và xác thực khi cần
        → AuthenticationManager
        → AuthenticationProvider phù hợp
        → Authentication đã xác thực
    → SecurityContext chứa kết quả xác thực
    → Kiểm tra quyền truy cập HTTP request
    → Controller / Service và kiểm tra quyền method nếu cấu hình
    → Response
```

Sơ đồ thể hiện luồng khái quát, không phải danh sách cố định của mọi filter. Request đã có session hợp lệ có thể tải lại context; request công khai có thể không cần xác thực. Đăng nhập form thành công thường kết thúc bằng redirect thay vì đi thẳng đến controller nghiệp vụ trong cùng request.

| Giai đoạn | Thành phần chính | Vai trò |
| --- | --- | --- |
| 1. Client request | Trình duyệt hoặc API client | Gửi thông tin đăng nhập, cookie phiên hoặc token. |
| 2. Filter chain | `FilterChainProxy`, `SecurityFilterChain` | Chọn chain và chạy các filter bảo mật trước controller. |
| 3. Điều phối xác thực | `AuthenticationManager`, thường là `ProviderManager` | Giao xử lý cho provider hỗ trợ kiểu token. |
| 4. Thực hiện xác thực | Provider cho username/password, JWT hoặc OAuth2/OIDC | Kiểm tra thông tin đầu vào và tạo kết quả xác thực. |
| 5. Security Context | `SecurityContext`, `SecurityContextHolder`, repository phù hợp | Cung cấp thông tin xác thực hiện tại và lưu/tải lại nếu cấu hình yêu cầu. |
| 6. Authorization | `AuthorizationFilter`, `AuthorizationManager`, method interceptor | Kiểm tra quyền truy cập request hoặc method. |
| 7. Controller / Service | Các thành phần nghiệp vụ của ứng dụng | Xử lý yêu cầu sau khi vượt qua các kiểm tra áp dụng. |
