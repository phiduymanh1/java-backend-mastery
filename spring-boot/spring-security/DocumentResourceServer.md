# OAuth2 Resource Server

Đọc sau [Spring Security](DocumentSpringSecurity.md) và [JWT](DocumentJWT.md). Phần đăng nhập được trình bày riêng trong [OIDC](DocumentOIDC.md).

## 1. Các vai trò trong hệ thống

| Thành phần | Trách nhiệm | Ví dụ |
| --- | --- | --- |
| Authorization Server / OpenID Provider (OP) | Xác thực người dùng, cấp token theo giao thức đã cấu hình. | Hệ thống định danh của tổ chức. |
| OAuth2 Client / OIDC Relying Party (RP) | Bắt đầu đăng nhập, nhận callback và đổi code lấy token. | Ứng dụng web Spring Boot. |
| Resource Server | Nhận access token, xác thực token và bảo vệ tài nguyên. | REST API đơn hàng. |
| Resource Owner | Chủ thể cho phép truy cập tài nguyên. | Người dùng. |

Một ứng dụng có thể đảm nhiệm nhiều vai trò. Bật Resource Server không tự tạo trang đăng nhập, endpoint cấp token hay refresh token.

OAuth2 cung cấp cơ chế ủy quyền. OIDC bổ sung lớp danh tính bằng scope `openid` và ID token. JWT chỉ là định dạng token: access token có thể là JWT hoặc opaque token.

## 2. Phân biệt các token

| Token | Bên sử dụng | Mục đích |
| --- | --- | --- |
| Access token | Resource Server | Cho phép gọi API theo audience, scope và chính sách của API. |
| ID token | OIDC Client | Xác nhận kết quả đăng nhập của người dùng với client. |
| Refresh token | Client gửi tới token endpoint | Xin token mới theo chính sách của bên cấp. |

API nghiệp vụ nhận **access token**. Không gửi ID token hoặc refresh token thay access token. `aud` của ID token thường là client ID; `aud` của access token phải phù hợp API nhận nó. Một token có chữ ký hợp lệ chưa đủ để kết luận nó đúng mục đích.

## 3. Logic Resource Server

```text
Client → API: Authorization: Bearer <access-token>
  → BearerTokenAuthenticationFilter đọc token
  → AuthenticationManager / ProviderManager
  → JwtAuthenticationProvider
      → JwtDecoder xác thực chữ ký và claims
      → JwtAuthenticationConverter tạo authorities
  → JwtAuthenticationToken được đặt vào SecurityContext
  → Kiểm tra quyền HTTP / @PreAuthorize
  → Controller → Service
```

Token không hợp lệ dẫn đến 401; token hợp lệ nhưng thiếu quyền dẫn đến 403. Thiếu token ở endpoint bắt buộc xác thực cũng dẫn đến 401 với cấu hình Bearer bên dưới. `permitAll()` không bỏ qua quá trình xác thực khi client đã gửi một Bearer token lỗi.

Resource Server phải tin cậy issuer và khóa đã cấu hình, kiểm tra chữ ký, thời hạn, audience và claims bắt buộc theo hợp đồng token của hệ thống. JWT xác minh cục bộ không tự biết tài khoản vừa bị khóa hay token vừa bị thu hồi; xem thêm [vòng đời token](DocumentJWT.md).

### 3.1 Dependency và cấu hình

Thêm vào `pom.xml` của ứng dụng API tương lai:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

Ví dụ `application.yml`; thay issuer bằng URL thật do bên cấp token công bố:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
app:
  security:
    audience: orders-api
```

Issuer hỗ trợ discovery để tìm JWKS chứa public keys. API không cần private key của bên cấp token hay client secret để xác minh JWT bất đối xứng. Thời điểm tải metadata/khóa và cơ chế cache phụ thuộc phiên bản, cấu hình; không phải mỗi request đều gọi bên cấp token.

### 3.2 Cấu hình API và audience

Ví dụ lớp `ApiSecurityConfig.java` cho **ứng dụng API riêng**. Quy tắc quyền minh họa dùng scope `orders.read` và `orders.write` đã được bên cấp đưa vào access token.

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.JwtValidators;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity
public class ApiSecurityConfig {
    @Bean
    SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public/**").permitAll()
                        .requestMatchers(HttpMethod.GET, "/api/orders/**")
                            .hasAuthority("SCOPE_orders.read")
                        .requestMatchers(HttpMethod.POST, "/api/orders/**")
                            .hasAuthority("SCOPE_orders.write")
                        .anyRequest().denyAll())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
                .build();
    }

    @Bean
    JwtDecoder jwtDecoder(
            @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}") String issuer,
            @Value("${app.security.audience}") String audience) {
        NimbusJwtDecoder decoder = NimbusJwtDecoder.withIssuerLocation(issuer).build();
        OAuth2TokenValidator<Jwt> audienceValidator = jwt ->
                jwt.getAudience() != null && jwt.getAudience().contains(audience)
                        ? OAuth2TokenValidatorResult.success()
                        : OAuth2TokenValidatorResult.failure(new OAuth2Error(
                                "invalid_token", "Access token has an invalid audience", null));
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
                JwtValidators.createDefaultWithIssuer(issuer), audienceValidator));
        return decoder;
    }
}
```

`withIssuerLocation(...)` trong ví dụ yêu cầu Spring Security 6.2 trở lên. Bean decoder tự khai báo phải giữ các validator cần thiết; chỉ gắn audience validator sẽ làm mất kiểm tra mặc định tương ứng. Nếu hợp đồng yêu cầu claim luôn hiện diện, như `exp`, thêm validator bắt buộc claim thay vì chỉ kiểm tra thời hạn khi claim có mặt.

Tắt CSRF ở đây chỉ phù hợp API nhận Bearer header và không dựa vào cookie/session tự gửi của trình duyệt để xác thực. Nếu đổi cách truyền token sang cookie, cần thiết kế lại bảo vệ CSRF. CORS cho frontend khác origin được cấu hình riêng.

Spring mặc định chuyển scope thành authority có tiền tố `SCOPE_`. Claim tùy biến `roles` hoặc `realm_access.roles` cần converter riêng theo cấu trúc thực tế; không tự suy ra `ROLE_ADMIN`. Ngoài quyền scope, service phải kiểm tra quyền sở hữu dữ liệu, ví dụ người dùng có được xem đơn hàng đó hay không.

### 3.3 Khi access token là opaque token

Thay cơ chế `jwt(...)` bằng `opaqueToken(...)` và cấu hình introspection endpoint cùng thông tin client được phép truy vấn. API hỏi bên cấp về trạng thái token, thay vì xác minh chữ ký JWT cục bộ. Không bật hai cơ chế như thể mọi token đều được thử lần lượt; cần chọn hoặc điều phối theo thiết kế cụ thể.

## 4. Các tình huống cần kiểm tra khi triển khai

| Tình huống | Kết quả mong đợi |
| --- | --- |
| Gọi API được bảo vệ không có token | 401, không chuyển hướng sang OP. |
| Token sai chữ ký, hết hạn, sai issuer hoặc audience | 401. |
| Access token hợp lệ, thiếu scope | 403. |
| Access token hợp lệ, đúng scope và quyền dữ liệu | Xử lý nghiệp vụ. |
| Gửi ID token vào API | Bị từ chối theo audience và hợp đồng access token. |

Đây là tiêu chí kiểm chứng khi triển khai, chưa phải kết quả test của repo tài liệu.



## 5. Nguồn đối chiếu

- [Spring Security: JWT Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html): decoder, validator và authorities.

Tài liệu Spring ở đường dẫn không khóa phiên bản có thể cập nhật. Khi triển khai, đối chiếu lại với phiên bản Spring Boot/Security thực tế của ứng dụng.
