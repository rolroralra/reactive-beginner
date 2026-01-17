# Spring Security 통합

## 1. 개요

Spring Security는 WebFlux 환경에서도 리액티브 방식으로 인증과 인가를 처리합니다. `spring-boot-starter-security`를 사용하여 손쉽게 보안을 구성할 수 있습니다.

## 2. 의존성

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-security'

    // JWT 사용 시
    implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.3'
}
```

## 3. 기본 설정

### 3.1 SecurityWebFilterChain 설정

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/auth/**").permitAll()
                .pathMatchers("/admin/**").hasRole("ADMIN")
                .pathMatchers("/api/**").authenticated()
                .anyExchange().authenticated()
            )
            .httpBasic(httpBasic -> {})
            .formLogin(formLogin -> formLogin.disable())
            .build();
    }
}
```

### 3.2 사용자 정보 설정

```java
@Bean
public MapReactiveUserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder().encode("password"))
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder().encode("admin"))
        .roles("USER", "ADMIN")
        .build();

    return new MapReactiveUserDetailsService(user, admin);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

## 4. JWT 인증

### 4.1 JWT 서비스

```java
package com.example.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.List;

@Service
public class JwtService {

    private final SecretKey key;
    private final long expiration;

    public JwtService(
            @Value("${jwt.secret}") String secret,
            @Value("${jwt.expiration:86400000}") long expiration) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes());
        this.expiration = expiration;
    }

    public String generateToken(String username, List<String> roles) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .subject(username)
            .claim("roles", roles)
            .issuedAt(now)
            .expiration(expiryDate)
            .signWith(key)
            .compact();
    }

    public Claims validateToken(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public String getUsernameFromToken(String token) {
        return validateToken(token).getSubject();
    }

    @SuppressWarnings("unchecked")
    public List<String> getRolesFromToken(String token) {
        return (List<String>) validateToken(token).get("roles");
    }
}
```

### 4.2 JWT 인증 필터

```java
package com.example.security;

import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.stream.Collectors;

@Component
public class JwtAuthenticationFilter implements WebFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return chain.filter(exchange);
        }

        String token = authHeader.substring(7);

        try {
            String username = jwtService.getUsernameFromToken(token);
            List<String> roles = jwtService.getRolesFromToken(token);

            List<SimpleGrantedAuthority> authorities = roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());

            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(username, null, authorities);

            return chain.filter(exchange)
                .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
        } catch (Exception e) {
            return chain.filter(exchange);
        }
    }
}
```

### 4.3 JWT Security 설정

```java
@Configuration
@EnableWebFluxSecurity
public class JwtSecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    public JwtSecurityConfig(JwtAuthenticationFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .httpBasic(httpBasic -> httpBasic.disable())
            .formLogin(formLogin -> formLogin.disable())
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/auth/**").permitAll()
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .addFilterAt(jwtFilter, SecurityWebFiltersOrder.AUTHENTICATION)
            .build();
    }
}
```

### 4.4 인증 컨트롤러

```java
package com.example.controller;

import com.example.security.JwtService;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final JwtService jwtService;
    private final ReactiveUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public AuthController(
            JwtService jwtService,
            ReactiveUserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @PostMapping("/login")
    public Mono<Map<String, String>> login(@RequestBody LoginRequest request) {
        return userDetailsService.findByUsername(request.username())
            .filter(user -> passwordEncoder.matches(request.password(), user.getPassword()))
            .map(user -> {
                List<String> roles = user.getAuthorities().stream()
                    .map(a -> a.getAuthority().replace("ROLE_", ""))
                    .toList();

                String token = jwtService.generateToken(user.getUsername(), roles);
                return Map.of("token", token);
            })
            .switchIfEmpty(Mono.error(
                new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Invalid credentials")));
    }
}

record LoginRequest(String username, String password) {}
```

## 5. 사용자 정보 접근

### 5.1 Controller에서 접근

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/me")
    public Mono<Map<String, Object>> getCurrentUser(
            @AuthenticationPrincipal Mono<UserDetails> principal) {
        return principal.map(user -> Map.of(
            "username", user.getUsername(),
            "authorities", user.getAuthorities()
        ));
    }

    // 또는 직접 Authentication 사용
    @GetMapping("/profile")
    public Mono<Map<String, Object>> getProfile() {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .map(auth -> Map.of(
                "username", auth.getName(),
                "authorities", auth.getAuthorities()
            ));
    }
}
```

### 5.2 Service에서 접근

```java
@Service
public class UserService {

    public Mono<String> getCurrentUsername() {
        return ReactiveSecurityContextHolder.getContext()
            .map(context -> context.getAuthentication().getName());
    }

    public Mono<Boolean> hasRole(String role) {
        return ReactiveSecurityContextHolder.getContext()
            .map(context -> context.getAuthentication().getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_" + role)));
    }
}
```

## 6. 메서드 레벨 보안

### 6.1 설정

```java
@Configuration
@EnableReactiveMethodSecurity
public class MethodSecurityConfig {
}
```

### 6.2 어노테이션 사용

```java
@Service
public class SecuredService {

    @PreAuthorize("hasRole('USER')")
    public Mono<String> userMethod() {
        return Mono.just("User method executed");
    }

    @PreAuthorize("hasRole('ADMIN')")
    public Mono<String> adminMethod() {
        return Mono.just("Admin method executed");
    }

    @PreAuthorize("#username == authentication.name")
    public Mono<User> getUser(String username) {
        return userRepository.findByUsername(username);
    }

    @PostAuthorize("returnObject.username == authentication.name")
    public Mono<User> getUserById(Long id) {
        return userRepository.findById(id);
    }
}
```

## 7. 커스텀 인증

### 7.1 ReactiveAuthenticationManager

```java
@Component
public class CustomAuthenticationManager implements ReactiveAuthenticationManager {

    private final ReactiveUserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    public CustomAuthenticationManager(
            ReactiveUserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        this.userDetailsService = userDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public Mono<Authentication> authenticate(Authentication authentication) {
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();

        return userDetailsService.findByUsername(username)
            .filter(user -> passwordEncoder.matches(password, user.getPassword()))
            .map(user -> new UsernamePasswordAuthenticationToken(
                user.getUsername(),
                null,
                user.getAuthorities()))
            .switchIfEmpty(Mono.error(
                new BadCredentialsException("Invalid credentials")));
    }
}
```

### 7.2 데이터베이스 기반 UserDetailsService

```java
@Service
public class DatabaseUserDetailsService implements ReactiveUserDetailsService {

    private final UserRepository userRepository;

    public DatabaseUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public Mono<UserDetails> findByUsername(String username) {
        return userRepository.findByUsername(username)
            .map(user -> User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .roles(user.getRoles().toArray(new String[0]))
                .build());
    }
}
```

## 8. CORS 설정

```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
    return http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(csrf -> csrf.disable())
        // ... 기타 설정
        .build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

## 9. OAuth2 / OIDC

### 9.1 의존성

```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
```

### 9.2 OAuth2 클라이언트 설정

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
```

```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
    return http
        .authorizeExchange(exchanges -> exchanges
            .pathMatchers("/", "/public/**").permitAll()
            .anyExchange().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .authenticationSuccessHandler(successHandler())
        )
        .build();
}
```

### 9.3 Resource Server (JWT)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-issuer.com
          # 또는
          jwk-set-uri: https://your-issuer.com/.well-known/jwks.json
```

```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
    return http
        .authorizeExchange(exchanges -> exchanges
            .pathMatchers("/public/**").permitAll()
            .anyExchange().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
        )
        .build();
}

@Bean
public ReactiveJwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthoritiesClaimName("roles");
    authoritiesConverter.setAuthorityPrefix("ROLE_");

    ReactiveJwtAuthenticationConverter converter = new ReactiveJwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(
        new ReactiveJwtGrantedAuthoritiesConverterAdapter(authoritiesConverter));
    return converter;
}
```

## 10. 테스트

```java
@WebFluxTest(UserController.class)
@Import(SecurityConfig.class)
class SecuredControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private UserService userService;

    @Test
    @WithMockUser(username = "user", roles = "USER")
    void shouldAccessUserEndpoint() {
        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void shouldAccessAdminEndpoint() {
        webTestClient.get()
            .uri("/admin/users")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void shouldDenyWithoutAuth() {
        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isUnauthorized();
    }
}
```

## 11. 다음 단계

- [R2DBC](12_R2DBC.md): 데이터베이스 통합
- [Testing](10_Testing.md): 보안 테스트
- [Filter와 Interceptor](09_Filter와_Interceptor.md): 커스텀 보안 필터
