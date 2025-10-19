---
title: "Spring Boot: Từ 'Zero' đến 'Hero' API"
meta_title: ""
description: "this is meta description"
date: 2025-05-04T05:00:00Z
image: "/images/gallery/05.jpg"
categories: ["Technology", "Data", "Java"]
author: "Lê Anh Tiến"
tags: ["java", "technology"]
draft: false
---
---
Spring Boot: Từ "Zero" đến "Hero" API: Một series "cầm tay chỉ việc". Bắt đầu từ File > New Project cho đến khi "deploy" được một con API RESTful hoàn chỉnh. Tập trung vào: Spring Data JPA (làm việc với database "dễ như ăn kẹo") và Spring Security (bảo mật "tận răng").

---

# Spring Boot: Từ “Zero” đến “Hero” API

> Mục tiêu: Xây dựng một API quản lý sản phẩm có xác thực/phan quyền bằng JWT, lưu dữ liệu PostgreSQL, migration bằng Flyway, test bằng Testcontainers, đóng gói Docker và triển khai.

## Yêu cầu môi trường

* Java 21 (hoặc 17)
* Maven 3.9+ hoặc Gradle 8+
* Docker Desktop (để chạy Postgres/Testcontainers/Docker build)
* IDE: IntelliJ IDEA (khuyến nghị)

---

## Phần 0 — Khởi tạo dự án (File > New Project)

**Cách 1: Spring Initializr (khuyên dùng)**

* Project: *Maven*
* Language: *Java*
* Spring Boot: *3.x mới nhất*
* Dependencies:

  * Spring Web
  * Validation
  * Spring Data JPA
  * PostgreSQL Driver
  * Flyway
  * Spring Security
  * Lombok (tuỳ chọn)
  * springdoc-openapi-starter-webmvc-ui (Swagger UI)
  * Testcontainers (JUnit, PostgreSQL)

**Cấu trúc khởi điểm**

```
src
 └─ main
    ├─ java/com/example/heroapi/...
    └─ resources
       ├─ application.yml
       └─ db/migration        # Flyway
```

**`pom.xml` tối thiểu (rút gọn các phần chính)**:

```xml
<project>
  <properties>
    <java.version>21</java.version>
    <springdoc.version>2.6.0</springdoc.version>
  </properties>

  <dependencies>
    <!-- Web, Validation -->
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- JPA + DB -->
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId><artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>org.flywaydb</groupId><artifactId>flyway-core</artifactId>
    </dependency>

    <!-- Security -->
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId><artifactId>jjwt-api</artifactId><version>0.11.5</version>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId><artifactId>jjwt-impl</artifactId><version>0.11.5</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId><artifactId>jjwt-jackson</artifactId><version>0.11.5</version>
      <scope>runtime</scope>
    </dependency>

    <!-- Swagger UI -->
    <dependency>
      <groupId>org.springdoc</groupId><artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>${springdoc.version}</version>
    </dependency>

    <!-- Dev helper -->
    <dependency>
      <groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId><artifactId>junit-jupiter</artifactId><scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId><artifactId>postgresql</artifactId><scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

**`application.yml` đa profile**

```yaml
spring:
  application:
    name: hero-api
  datasource:
    url: jdbc:postgresql://localhost:5432/hero
    username: hero
    password: hero
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true

server:
  port: 8080

logging:
  level:
    org.springframework: info

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:postgresql://localhost:5433/hero_dev
    username: hero
    password: hero
```

**Docker chạy Postgres cho dev (docker-compose)**

```yaml
version: "3.8"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: hero_dev
      POSTGRES_USER: hero
      POSTGRES_PASSWORD: hero
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

Chạy: `docker compose up -d`

---

## Phần 1 — “Hello API” và Swagger

**Controller mẫu**

```java
@RestController
@RequestMapping("/api/health")
public class HealthController {
  @GetMapping
  public Map<String, Object> health() {
    return Map.of("status", "OK", "service", "hero-api");
  }
}
```

**Swagger/OpenAPI**
Sau khi chạy ứng dụng:

* OpenAPI JSON: `/v3/api-docs`
* Swagger UI: `/swagger-ui.html`

---

## Phần 2 — Spring Data JPA: Mô hình dữ liệu và migration

### 2.1 Thiết kế domain

Ví dụ hai thực thể: `User` và `Product` (một user có thể tạo nhiều sản phẩm).

**Flyway migration: `src/main/resources/db/migration/V1__init.sql`**

```sql
create table users (
  id bigserial primary key,
  email varchar(255) not null unique,
  password varchar(255) not null,
  role varchar(50) not null,
  created_at timestamptz default now()
);

create table products (
  id bigserial primary key,
  name varchar(255) not null,
  description text,
  price numeric(12,2) not null,
  owner_id bigint not null references users(id),
  created_at timestamptz default now()
);

create index idx_products_owner on products(owner_id);
```

**Entity và Repository**

```java
@Entity @Table(name="users")
@Getter @Setter
public class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  @Column(nullable=false, unique=true) private String email;
  @Column(nullable=false) private String password;
  @Column(nullable=false) @Enumerated(EnumType.STRING) private Role role;
  @Column(name="created_at") private Instant createdAt = Instant.now();
  public enum Role { ADMIN, USER }
}

public interface UserRepository extends JpaRepository<User, Long> {
  Optional<User> findByEmail(String email);
}

@Entity @Table(name="products")
@Getter @Setter
public class Product {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  @Column(nullable=false) private String name;
  private String description;
  @Column(nullable=false) private BigDecimal price;
  @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name="owner_id", nullable=false)
  private User owner;
  @Column(name="created_at") private Instant createdAt = Instant.now();
}

public interface ProductRepository extends JpaRepository<Product, Long>, JpaSpecificationExecutor<Product> {
  Page<Product> findByOwnerId(Long ownerId, Pageable pageable);
}
```

---

## Phần 3 — DTO, Validation, Service, Controller

**DTO và Validation**

```java
public record ProductCreateRequest(
  @NotBlank String name,
  String description,
  @NotNull @DecimalMin("0.0") BigDecimal price
) {}

public record ProductResponse(
  Long id, String name, String description, BigDecimal price, Long ownerId, Instant createdAt
) {
  public static ProductResponse of(Product p) {
    return new ProductResponse(
      p.getId(), p.getName(), p.getDescription(), p.getPrice(),
      p.getOwner().getId(), p.getCreatedAt()
    );
  }
}
```

**Service**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class ProductService {
  private final ProductRepository repo;
  private final UserRepository users;

  public ProductResponse create(Long ownerId, ProductCreateRequest req) {
    User owner = users.findById(ownerId).orElseThrow(() -> new NotFoundException("user"));
    Product p = new Product();
    p.setName(req.name());
    p.setDescription(req.description());
    p.setPrice(req.price());
    p.setOwner(owner);
    return ProductResponse.of(repo.save(p));
  }

  @Transactional(readOnly = true)
  public Page<ProductResponse> list(Long ownerId, Pageable pageable) {
    return repo.findByOwnerId(ownerId, pageable).map(ProductResponse::of);
  }

  @Transactional(readOnly = true)
  public ProductResponse get(Long id) {
    return repo.findById(id).map(ProductResponse::of).orElseThrow(() -> new NotFoundException("product"));
  }

  public void delete(Long id) { repo.deleteById(id); }
}
```

**Exception handling**

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class NotFoundException extends RuntimeException {
  public NotFoundException(String what) { super(what + " not found"); }
}

@RestControllerAdvice
public class ApiExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<?> handleValidation(MethodArgumentNotValidException ex) {
    var errors = ex.getBindingResult().getFieldErrors().stream()
      .collect(Collectors.toMap(FieldError::getField, DefaultMessageSourceResolvable::getDefaultMessage,(a,b)->a));
    return ResponseEntity.badRequest().body(Map.of("error","validation", "details", errors));
  }

  @ExceptionHandler(NotFoundException.class)
  public ResponseEntity<?> handleNotFound(NotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("error","not_found","message", ex.getMessage()));
  }
}
```

**Controller**

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {
  private final ProductService service;

  @PostMapping("/{ownerId}")
  public ProductResponse create(@PathVariable Long ownerId, @Valid @RequestBody ProductCreateRequest req) {
    return service.create(ownerId, req);
  }

  @GetMapping("/owner/{ownerId}")
  public Page<ProductResponse> list(@PathVariable Long ownerId, Pageable pageable) {
    return service.list(ownerId, pageable);
  }

  @GetMapping("/{id}")
  public ProductResponse get(@PathVariable Long id) { return service.get(id); }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public void delete(@PathVariable Long id) { service.delete(id); }
}
```

---

## Phần 4 — Spring Security “tận răng” với JWT

### 4.1 Mật khẩu và người dùng

**CommandLineRunner khởi tạo admin**

```java
@Bean
CommandLineRunner seed(UserRepository users, PasswordEncoder encoder) {
  return args -> {
    users.findByEmail("admin@local")
      .orElseGet(() -> {
        User u = new User();
        u.setEmail("admin@local");
        u.setPassword(encoder.encode("admin123"));
        u.setRole(User.Role.ADMIN);
        return users.save(u);
      });
  };
}
```

**Bean `PasswordEncoder`**

```java
@Configuration
public class PasswordConfig {
  @Bean
  public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

### 4.2 JWT tiện ích và filter

**JwtService**

```java
@Service
public class JwtService {
  @Value("${app.jwt.secret}") private String secret;
  @Value("${app.jwt.exp}") private long expMillis;

  public String generate(String subject, Map<String, Object> claims) {
    var key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    var now = new Date();
    return Jwts.builder()
      .setClaims(claims)
      .setSubject(subject)
      .setIssuedAt(now)
      .setExpiration(new Date(now.getTime() + expMillis))
      .signWith(key, SignatureAlgorithm.HS256)
      .compact();
  }

  public Jws<Claims> parse(String token) {
    var key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    return Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
  }
}
```

**Filter trích xuất JWT**

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwt;
  private final UserRepository users;

  @Override
  protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    String auth = req.getHeader(HttpHeaders.AUTHORIZATION);
    if (auth != null && auth.startsWith("Bearer ")) {
      String token = auth.substring(7);
      try {
        var claims = jwt.parse(token).getBody();
        String email = claims.getSubject();
        var user = users.findByEmail(email).orElse(null);
        if (user != null) {
          var authorities = List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole().name()));
          var principal = new org.springframework.security.core.userdetails.User(email, user.getPassword(), authorities);
          var authentication = new UsernamePasswordAuthenticationToken(principal, null, authorities);
          SecurityContextHolder.getContext().setAuthentication(authentication);
        }
      } catch (Exception ignored) {}
    }
    chain.doFilter(req, res);
  }
}
```

### 4.3 Cấu hình Security (Spring Security 6+)

```java
@Configuration
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

  private final JwtAuthFilter jwtAuthFilter;

  @Bean
  SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/v3/api-docs/**","/swagger-ui.html","/swagger-ui/**","/api/auth/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
        .anyRequest().authenticated()
      )
      .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
  }

  @Bean
  AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
    return cfg.getAuthenticationManager();
  }
}
```

### 4.4 Đăng ký/đăng nhập

```java
public record AuthRequest(String email, String password) {}
public record AuthResponse(String token) {}

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
  private final UserRepository users;
  private final PasswordEncoder encoder;
  private final JwtService jwt;

  @PostMapping("/register")
  public AuthResponse register(@RequestBody @Valid AuthRequest req) {
    users.findByEmail(req.email()).ifPresent(u -> { throw new ResponseStatusException(HttpStatus.CONFLICT, "Email exists"); });
    User u = new User();
    u.setEmail(req.email());
    u.setPassword(encoder.encode(req.password()));
    u.setRole(User.Role.USER);
    users.save(u);
    String token = jwt.generate(u.getEmail(), Map.of("role", u.getRole().name(), "uid", u.getId()));
    return new AuthResponse(token);
  }

  @PostMapping("/login")
  public AuthResponse login(@RequestBody AuthRequest req) {
    User u = users.findByEmail(req.email()).orElseThrow(() -> new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Bad credentials"));
    if (!encoder.matches(req.password(), u.getPassword())) throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Bad credentials");
    String token = jwt.generate(u.getEmail(), Map.of("role", u.getRole().name(), "uid", u.getId()));
    return new AuthResponse(token);
  }
}
```

**Bảo vệ endpoint theo vai trò (ví dụ xoá sản phẩm, chỉ ADMIN)**

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void delete(@PathVariable Long id) { service.delete(id); }
```

---

## Phần 5 — Kiểm thử: Unit + Integration với Testcontainers

**Integration test khởi động Postgres container**

```java
@SpringBootTest
@Testcontainers
class ProductIntegrationTest {

  @Container
  static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16")
      .withDatabaseName("test")
      .withUsername("test")
      .withPassword("test");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
  }

  @Autowired ProductRepository repo;

  @Test
  void createAndFind() {
    // kiểm thử CRUD cơ bản...
  }
}
```

---

## Phần 6 — Actuator, Logging, và quan sát

Thêm dependency:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Cấu hình mở endpoint cơ bản:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

---

## Phần 7 — Đóng gói Docker và chạy

**Dockerfile (multi-stage)**

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml ./
RUN mvn -q -e -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

# Run stage
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0"
EXPOSE 8080
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar"]
```

**docker-compose cho app + db (prod-like)**

```yaml
version: "3.8"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: hero
      POSTGRES_USER: hero
      POSTGRES_PASSWORD: hero
    volumes: [pgdata:/var/lib/postgresql/data]
  app:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/hero
      SPRING_DATASOURCE_USERNAME: hero
      SPRING_DATASOURCE_PASSWORD: hero
      APP_JWT_SECRET: "change_this_very_long_secret_key"
      APP_JWT_EXP: "3600000"
    ports: ["8080:8080"]
    depends_on: [db]
volumes: { pgdata: {} }
```

**Lưu ý cấu hình biến môi trường**

```yaml
app:
  jwt:
    secret: ${APP_JWT_SECRET:dev_secret_please_change}
    exp: ${APP_JWT_EXP:3600000}
```

---

## Phần 8 — Triển khai

Các hướng triển khai phổ biến (chọn một):

* **Máy chủ/Docker VM**: build image, push registry, `docker compose up -d` trên server.
* **Render/Fly.io/Railway**: kết nối repository, cấu hình biến môi trường, add service Postgres, thiết lập health check.
* **Kubernetes**: tạo Deployment, Service, ConfigMap/Secret; dùng Postgres managed service; Ingress cho HTTP.

Các bước chung:

1. Thiết lập **biến môi trường** (DB, JWT secret, profile).
2. Đảm bảo **cổng 8080** mở và **health endpoint** hoạt động.
3. Áp dụng **migrate** Flyway tự động lúc start.
4. Bật **log** mức `info` cho Spring, `warn` cho Hibernate nếu cần.

---

## Phần 9 — Kiểm thử thủ công

**Đăng ký → Lấy JWT**

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"u1@local","password":"pass123"}'
```

**Tạo product (cần Bearer token)**

```bash
TOKEN=... # gán JWT nhận được
curl -X POST http://localhost:8080/api/products/1 \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Book","description":"Clean Architecture","price":350000}'
```

**Danh sách product theo owner (public GET)**

```bash
curl "http://localhost:8080/api/products/owner/1?page=0&size=10&sort=createdAt,desc"
```

---

## Phần 10 — Check-list “Hero”

* [ ] Swagger UI hiển thị đầy đủ contract.
* [ ] Validation trả lỗi rõ ràng (400).
* [ ] Auth/Role bảo vệ đúng endpoint (401/403).
* [ ] Log, Actuator, Health OK.
* [ ] Migration Flyway chạy khi khởi động.
* [ ] Testcontainers qua CI hoạt động xanh.
* [ ] Docker image khởi chạy, cấu hình bằng biến môi trường.
* [ ] Triển khai môi trường “prod-like” thành công.

---

### Ghi chú kiến trúc

* **Service/DTO** tách biệt Entity giúp an toàn và dễ refactor.
* **Open-in-view: false** để tránh n+1 và rò rỉ session; dùng `@Transactional(readOnly = true)` cho truy vấn.
* **Specification/QueryDSL** nếu cần lọc nâng cao.
* **Method Security** (`@PreAuthorize`) cho các quy tắc truy cập chi tiết.
* **Observability**: Actuator + Prometheus/Grafana khi cần theo dõi nâng cao.

---