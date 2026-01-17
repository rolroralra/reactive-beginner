# R2DBC (Reactive Relational Database Connectivity)

## 1. 개요

R2DBC는 **리액티브 관계형 데이터베이스 연결**을 위한 표준입니다. 기존 JDBC가 블로킹 방식인 반면, R2DBC는 완전한 Non-Blocking 데이터베이스 접근을 제공합니다.

### 1.1 JDBC vs R2DBC

| 특성 | JDBC | R2DBC |
|------|------|-------|
| I/O 모델 | Blocking | Non-Blocking |
| 리턴 타입 | Object, List | Mono, Flux |
| 트랜잭션 | 동기 | 리액티브 |
| Spring 통합 | JPA, JdbcTemplate | Spring Data R2DBC |

### 1.2 지원 데이터베이스

- PostgreSQL
- MySQL / MariaDB
- Microsoft SQL Server
- H2
- Oracle

## 2. 의존성

### 2.1 PostgreSQL

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    runtimeOnly 'org.postgresql:r2dbc-postgresql'

    // 테스트용
    testImplementation 'io.projectreactor:reactor-test'
    testRuntimeOnly 'io.r2dbc:r2dbc-h2'
}
```

### 2.2 MySQL

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    runtimeOnly 'io.asyncer:r2dbc-mysql'
}
```

### 2.3 H2 (테스트/개발용)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    runtimeOnly 'io.r2dbc:r2dbc-h2'
}
```

## 3. 설정

### 3.1 application.yml

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: user
    password: password
    pool:
      initial-size: 5
      max-size: 10
      max-idle-time: 30m

  # 스키마 초기화
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
      data-locations: classpath:data.sql
```

### 3.2 H2 설정 (개발/테스트)

```yaml
spring:
  r2dbc:
    url: r2dbc:h2:mem:///testdb;DB_CLOSE_DELAY=-1
    username: sa
    password:
```

### 3.3 Java 설정

```java
@Configuration
@EnableR2dbcRepositories
public class R2dbcConfig extends AbstractR2dbcConfiguration {

    @Override
    @Bean
    public ConnectionFactory connectionFactory() {
        return ConnectionFactories.get(ConnectionFactoryOptions.builder()
            .option(DRIVER, "postgresql")
            .option(HOST, "localhost")
            .option(PORT, 5432)
            .option(USER, "user")
            .option(PASSWORD, "password")
            .option(DATABASE, "mydb")
            .build());
    }
}
```

## 4. Entity 정의

### 4.1 기본 Entity

```java
package com.example.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.relational.core.mapping.Column;

import java.time.LocalDateTime;

@Table("users")
public class User {

    @Id
    private Long id;

    @Column("username")
    private String username;

    private String email;

    private String password;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    // 기본 생성자
    public User() {}

    public User(String username, String email, String password) {
        this.username = username;
        this.email = email;
        this.password = password;
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

### 4.2 Record 사용

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

@Table("products")
public record Product(
    @Id Long id,
    String name,
    String description,
    double price,
    int stock
) {
    // 새 엔티티용 팩토리 메서드
    public static Product create(String name, String description, double price, int stock) {
        return new Product(null, name, description, price, stock);
    }

    // ID 포함 복사
    public Product withId(Long id) {
        return new Product(id, name, description, price, stock);
    }
}
```

## 5. Repository

### 5.1 기본 Repository

```java
package com.example.repository;

import com.example.model.User;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Mono<User> findByUsername(String username);

    Mono<User> findByEmail(String email);

    Flux<User> findByUsernameContaining(String keyword);

    Mono<Boolean> existsByEmail(String email);
}
```

### 5.2 커스텀 쿼리

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    // 쿼리 메서드
    Flux<User> findByUsernameContainingIgnoreCase(String keyword);

    // @Query 어노테이션
    @Query("SELECT * FROM users WHERE email = :email")
    Mono<User> findByEmailCustom(String email);

    @Query("SELECT * FROM users WHERE created_at > :date ORDER BY created_at DESC")
    Flux<User> findRecentUsers(LocalDateTime date);

    @Modifying
    @Query("UPDATE users SET password = :password WHERE id = :id")
    Mono<Integer> updatePassword(Long id, String password);

    @Modifying
    @Query("DELETE FROM users WHERE created_at < :date")
    Mono<Integer> deleteOldUsers(LocalDateTime date);
}
```

### 5.3 ReactiveSortingRepository

```java
public interface UserRepository extends ReactiveSortingRepository<User, Long> {

    Flux<User> findAllBy(Sort sort);
}

// 사용
userRepository.findAllBy(Sort.by(Sort.Direction.DESC, "createdAt"));
```

### 5.4 페이징 지원

```java
public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Flux<User> findAllBy(Pageable pageable);

    @Query("SELECT * FROM users WHERE username LIKE :keyword")
    Flux<User> search(String keyword, Pageable pageable);
}

// 사용
Pageable pageable = PageRequest.of(0, 10, Sort.by("username"));
userRepository.findAllBy(pageable);
```

## 6. Service 레이어

```java
package com.example.service;

import com.example.model.User;
import com.example.repository.UserRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public Mono<User> findById(Long id) {
        return userRepository.findById(id);
    }

    public Mono<User> findByUsername(String username) {
        return userRepository.findByUsername(username);
    }

    public Flux<User> findAll() {
        return userRepository.findAll();
    }

    @Transactional
    public Mono<User> create(User user) {
        return userRepository.findByEmail(user.getEmail())
            .flatMap(existing -> Mono.<User>error(
                new IllegalArgumentException("Email already exists")))
            .switchIfEmpty(userRepository.save(user));
    }

    @Transactional
    public Mono<User> update(Long id, User user) {
        return userRepository.findById(id)
            .flatMap(existing -> {
                existing.setUsername(user.getUsername());
                existing.setEmail(user.getEmail());
                return userRepository.save(existing);
            });
    }

    @Transactional
    public Mono<Void> delete(Long id) {
        return userRepository.deleteById(id);
    }
}
```

## 7. 트랜잭션

### 7.1 어노테이션 기반

```java
@Service
public class TransferService {

    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;

    @Transactional
    public Mono<Void> transfer(Long fromId, Long toId, double amount) {
        return accountRepository.findById(fromId)
            .flatMap(from -> {
                if (from.getBalance() < amount) {
                    return Mono.error(new InsufficientBalanceException());
                }
                from.setBalance(from.getBalance() - amount);
                return accountRepository.save(from);
            })
            .then(accountRepository.findById(toId))
            .flatMap(to -> {
                to.setBalance(to.getBalance() + amount);
                return accountRepository.save(to);
            })
            .then(transactionRepository.save(
                new Transaction(fromId, toId, amount, LocalDateTime.now())))
            .then();
    }
}
```

### 7.2 프로그래밍 방식

```java
@Service
public class TransferService {

    private final TransactionalOperator transactionalOperator;

    public TransferService(TransactionalOperator transactionalOperator) {
        this.transactionalOperator = transactionalOperator;
    }

    public Mono<Void> transfer(Long fromId, Long toId, double amount) {
        return accountRepository.findById(fromId)
            .flatMap(from -> {
                // ... 전송 로직
            })
            .as(transactionalOperator::transactional);
    }
}
```

## 8. DatabaseClient (저수준 API)

```java
@Repository
public class CustomUserRepository {

    private final DatabaseClient databaseClient;

    public CustomUserRepository(DatabaseClient databaseClient) {
        this.databaseClient = databaseClient;
    }

    public Flux<User> findByCustomCriteria(String status, int minAge) {
        return databaseClient.sql("""
                SELECT * FROM users
                WHERE status = :status AND age >= :minAge
                ORDER BY username
            """)
            .bind("status", status)
            .bind("minAge", minAge)
            .map((row, metadata) -> new User(
                row.get("id", Long.class),
                row.get("username", String.class),
                row.get("email", String.class)
            ))
            .all();
    }

    public Mono<Long> insertUser(User user) {
        return databaseClient.sql("""
                INSERT INTO users (username, email, password)
                VALUES (:username, :email, :password)
            """)
            .bind("username", user.getUsername())
            .bind("email", user.getEmail())
            .bind("password", user.getPassword())
            .filter((statement, next) -> statement.returnGeneratedValues("id").execute())
            .map((row, metadata) -> row.get("id", Long.class))
            .one();
    }

    public Mono<Integer> batchInsert(List<User> users) {
        return Flux.fromIterable(users)
            .flatMap(user -> databaseClient.sql("""
                    INSERT INTO users (username, email) VALUES (:username, :email)
                """)
                .bind("username", user.getUsername())
                .bind("email", user.getEmail())
                .fetch()
                .rowsUpdated())
            .reduce(0, Integer::sum);
    }
}
```

## 9. 스키마 관리

### 9.1 schema.sql

```sql
-- src/main/resources/schema.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id);
```

### 9.2 Flyway 사용

```groovy
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-database-postgresql'  // PostgreSQL
```

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE
);

-- V2__add_password_column.sql
ALTER TABLE users ADD COLUMN password VARCHAR(255);
```

## 10. 연관 관계

R2DBC는 JPA와 달리 연관 관계를 자동으로 로드하지 않습니다. 수동으로 처리해야 합니다.

### 10.1 서비스에서 조인

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    public Mono<OrderWithUser> findOrderWithUser(Long orderId) {
        return orderRepository.findById(orderId)
            .flatMap(order -> userRepository.findById(order.getUserId())
                .map(user -> new OrderWithUser(order, user)));
    }

    public Flux<OrderWithUser> findUserOrders(Long userId) {
        return userRepository.findById(userId)
            .flatMapMany(user -> orderRepository.findByUserId(userId)
                .map(order -> new OrderWithUser(order, user)));
    }
}

record OrderWithUser(Order order, User user) {}
```

### 10.2 DatabaseClient로 조인

```java
public Flux<OrderWithUser> findAllOrdersWithUsers() {
    return databaseClient.sql("""
            SELECT o.*, u.username, u.email
            FROM orders o
            JOIN users u ON o.user_id = u.id
            ORDER BY o.created_at DESC
        """)
        .map((row, metadata) -> new OrderWithUser(
            new Order(
                row.get("id", Long.class),
                row.get("user_id", Long.class),
                row.get("total_amount", BigDecimal.class),
                row.get("status", String.class)
            ),
            new User(
                row.get("user_id", Long.class),
                row.get("username", String.class),
                row.get("email", String.class)
            )
        ))
        .all();
}
```

## 11. 테스트

### 11.1 @DataR2dbcTest

```java
@DataR2dbcTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUser() {
        User user = new User("testuser", "test@example.com", "password");

        StepVerifier.create(
            userRepository.save(user)
                .flatMap(saved -> userRepository.findById(saved.getId()))
        )
        .assertNext(found -> {
            assertThat(found.getUsername()).isEqualTo("testuser");
            assertThat(found.getEmail()).isEqualTo("test@example.com");
        })
        .verifyComplete();
    }

    @Test
    void shouldFindByUsername() {
        User user = new User("uniqueuser", "unique@example.com", "password");

        StepVerifier.create(
            userRepository.save(user)
                .then(userRepository.findByUsername("uniqueuser"))
        )
        .assertNext(found -> assertThat(found.getEmail()).isEqualTo("unique@example.com"))
        .verifyComplete();
    }
}
```

### 11.2 Testcontainers 사용

```groovy
testImplementation 'org.testcontainers:testcontainers'
testImplementation 'org.testcontainers:postgresql'
testImplementation 'org.testcontainers:r2dbc'
```

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () ->
            "r2dbc:postgresql://" + postgres.getHost() + ":" + postgres.getFirstMappedPort() +
            "/" + postgres.getDatabaseName());
        registry.add("spring.r2dbc.username", postgres::getUsername);
        registry.add("spring.r2dbc.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldWorkWithRealDatabase() {
        // 테스트 코드
    }
}
```

## 12. 다음 단계

- [Testing](10_Testing.md): R2DBC 테스트 상세
- [Spring Security 통합](11_Spring_Security_통합.md): 데이터베이스 기반 인증
- [WebClient](05_WebClient.md): HTTP 클라이언트
