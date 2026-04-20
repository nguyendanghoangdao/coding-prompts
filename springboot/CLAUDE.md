# Spring Boot Project Conventions

> This document defines the conventions, patterns, and standards for working with this Spring Boot project.
> AI agents must follow these rules when reading, generating, or modifying code.

---

## 1. Tech Stack

| Category          | Technology                        |
|-------------------|-----------------------------------|
| Language          | Java 21                           |
| Framework         | Spring Boot 3.5.13                |
| Model Mapper      | MapStruct                         |
| ORM               | Spring data JPA                   |
| Database          | Postgres                          |
| API Client        | OpenFeign                         |
| API Validation    | Spring Validation                 |
| API Documentation | Springdoc OpenAPI                 |
| Utilities         | Lombok, Apache Commons Lang3      |

---

## 2. Project Structure

```
src/
├── main/
│   ├── java/com/company/project/
│   │   ├── config/          # Spring configuration classes (@Configuration)
│   │   │   └── ApplicationConfig.java  # Main app config (@Component + @ConfigurationProperties(prefix = "application"))
│   │   ├── controller/      # REST controllers (@RestController)
│   │   ├── service/         # Business logic (@Service)
│   │   ├── repository/      # Data access layer (@Repository)
│   │   ├── entity/          # JPA entities (@Entity)
│   │   ├── dto/             # Data Transfer Objects (request/response)
│   │   ├── mapper/          # DTO <-> Entity mappers (MapStruct)
│   │   ├── exception/       # Custom exceptions & global handler
│   │   ├── security/        # Security config, filters, JWT utils
│   │   └── util/            # Utility/helper classes
│   └── resources/
│       ├── application.yml          # Base config
│       ├── application-dev.yml      # Dev profile
│       └── application-prod.yml     # Prod profile
└── test/
    └── java/com/company/project/
        ├── controller/      # Controller integration tests
        ├── service/         # Service unit tests
        └── repository/      # Repository slice tests
```

---

## 3. Architecture Overview

The application follows a **layered architecture**:

```
HTTP Request
     │
     ▼
┌─────────────┐
│  Controller │  ← Validates request, maps entity → DTO, returns BaseResponse<T>
└──────┬──────┘
       │ calls
       ▼
┌─────────────┐
│   Service   │  ← Business logic, returns entity directly
└──────┬──────┘
       │ calls
       ▼
┌─────────────┐
│ Repository  │  ← Data access only (JPA / Spring Data)
└─────────────┘
```
---

## 4. Controller Layer

### Responsibilities
- Define all API endpoints following RESTful conventions.
- Validate incoming request data using `@Valid`.
- Delegate all business logic to the service layer.
- Map entities returned from the service layer to response DTOs.
- Wrap all responses in `BaseResponse<T>`.

### API Design Rules
- Base path: `/api/v{n}/` (e.g., `/api/v1/users`)
- HTTP methods:
  - `GET` → read (idempotent)
  - `POST` → create
  - `PUT` → full update
  - `PATCH` → partial update
  - `DELETE` → remove
- HTTP status codes: `200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `500 Internal Server Error`

### Request Rules
All API requests that have a body **or** are GET requests with query params **must** be wrapped in a dedicated class. Path variables are the only exception.

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;
    private final UserMapper userMapper;

    // POST with body → wrapped in CreateUserRequest
    @PostMapping
    public ResponseEntity<BaseResponse<UserResponse>> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(BaseResponse.success(userMapper.toResponse(user)));
    }

    // GET with query params → wrapped in ListUsersRequest
    @GetMapping
    public ResponseEntity<BaseResponse<PageResponse<UserResponse>>> getUsers(
            @Valid ListUsersRequest request) {
        Page<User> page = userService.findAll(request);
        return ResponseEntity.ok(BaseResponse.success(userMapper.toPageResponse(page)));
    }

    // DELETE with path variable → path variable used directly
    @DeleteMapping("/{id}")
    public ResponseEntity<BaseResponse<Void>> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.ok(BaseResponse.success(null));
    }
}
```

---

## 5. Service Layer

### Responsibilities
- Implement all business logic.
- Interact with the repository layer for data access.
- **Return entities directly to the controller** — do NOT map to DTOs inside the service. This keeps services clean, reusable, and focused on business logic.
- Throw appropriate custom exceptions when validation or business rules fail.

```java
public interface UserService {
    User create(CreateUserRequest request);
    Page<User> findAll(ListUsersRequest request);
    void delete(Long id);
}

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    @Override
    @Transactional
    public User create(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new ValidationException("Email already in use");
        }
        User user = new User();
        user.setEmail(request.getEmail());
        // ... set other fields
        return userRepository.save(user);
    }

    @Override
    public Page<User> findAll(ListUsersRequest request) {
        Pageable pageable = PageRequest.of(request.getPage() - 1, request.getSize());
        return userRepository.findAll(pageable);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new ValidationException("User not found with id: " + id));
        userRepository.delete(user);
    }
}
```

---

## 6. Repository Layer

- Extends `JpaRepository<Entity, ID>`.
- Custom JPQL queries use `@Query` with `@Param`.
- No business logic — data access only.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findAllByStatus(@Param("status") UserStatus status);
}
```

---

## 7. Entity Layer

- All entities **must** extend `BaseEntity`.
- Annotate with `@Entity` and `@Table(name = "table_name")`.
- Use `UUID` as the primary key type with `@UuidGenerator`.
- Use Lombok: `@Getter`, `@Setter`, `@NoArgsConstructor`. **Do NOT use `@Data` on entities** (causes issues with `equals/hashCode` and Hibernate lazy loading).
- Entities that support soft delete must add `deletedAt` and `deletedBy` fields themselves — these are **not** part of `BaseEntity`.

### BaseEntity

```java
@MappedSuperclass
@Data
public class BaseEntity {

    @Column(name = "created_at", nullable = false, updatable = false)
    protected Instant createdAt;

    @Column(name = "updated_at")
    protected Instant updatedAt;

    @Column(name = "created_by")
    protected String createdBy;

    @Column(name = "updated_by")
    protected String updatedBy;
}
```

### Entity Example (with soft delete)

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
public class User extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserStatus status;

    // Soft delete — added manually, not inherited from BaseEntity
    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Column(name = "deleted_by")
    private String deletedBy;
}
```

---

## 8. DTO Layer

### Request DTOs
- Use Lombok `@Data`, `@AllArgsConstructor`, `@NoArgsConstructor`.
- Apply validation annotations (`@NotNull`, `@NotBlank`, `@Email`, `@Size`, etc.) for input validation.
- All paginated list requests must extend `PaginatedRequest`.

```java
// Base paginated request
@Data
@AllArgsConstructor
@NoArgsConstructor
public class PaginatedRequest {
    protected int page = 1;
    protected int size = 10;
}

// Paginated list request
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ListUsersRequest extends PaginatedRequest {
    private String name;
    private String email;
}

// Create request
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CreateUserRequest {
    @NotBlank
    @Email
    private String email;

    @NotBlank
    @Size(min = 8)
    private String password;

    @NotBlank
    private String fullName;
}
```

### Response DTOs
- Must never expose sensitive fields (e.g., passwords, tokens).
- Use Lombok `@Data`, `@AllArgsConstructor`, `@NoArgsConstructor`.

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class UserResponse {
    private Long id;
    private String email;
    private String fullName;
    private UserStatus status;
    private Instant createdAt;
}
```

### Common Response Wrappers

All API responses must be wrapped in `BaseResponse<T>`:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class BaseResponse<T> {
    private int code;
    private String message;
    private T data;
}
```

For paginated responses, `T` is `PageResponse<T>`:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class PageResponse<T> {
    private List<T> items;
    private int page;
    private int size;
    private long totalItems;
    private int totalPages;
}
```

---

## 9. Model Mapping (MapStruct)

- All mapping between entities and DTOs is handled in the **controller layer**.
- Mappers are placed in the `mapper` package.
- Mappers are interfaces annotated with `@Mapper(componentModel = "spring")`.

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);
    PageResponse<UserResponse> toPageResponse(Page<User> page);
}
```

> **Rule**: Services return entities. Controllers call mappers. Mappers never live in services, except for special case

---

## 10. Exception Handling

- All exceptions are handled globally by a single `@RestControllerAdvice` class.
- All error responses follow the `BaseResponse` format with an appropriate HTTP status code.

### Predefined Exception Types

| Exception             | Usage                                                                              | HTTP Status       |
|-----------------------|------------------------------------------------------------------------------------|-------------------|
| `ValidationException` | Business rule violations (e.g., duplicate email, entity not found before update/delete) | `400 Bad Request` |

Additional custom exceptions can be defined as needed. Each must be handled in the global handler with an appropriate status code.

```java
// Custom exception
public class ValidationException extends RuntimeException {
    public ValidationException(String message) {
        super(message);
    }
}

// Global handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<BaseResponse<Void>> handleValidation(ValidationException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new BaseResponse<>(400, ex.getMessage(), null));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<BaseResponse<Void>> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new BaseResponse<>(400, message, null));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<BaseResponse<Void>> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new BaseResponse<>(500, "Internal server error", null));
    }
}
```

---

## 11. Configuration

- Use `application.yml` (not `.properties`).
- The main application configuration class is `ApplicationConfig`, annotated with `@Component` and `@ConfigurationProperties(prefix = "application")`.
- Secrets must come from environment variables — never hardcoded.

```yaml
# application.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

application:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 86400000
```

```java
@Component
@ConfigurationProperties(prefix = "application")
@Data
public class ApplicationConfig {
    private Jwt jwt;

    @Data
    public static class Jwt {
        private String secret;
        private long expirationMs;
    }
}
```

---

## 12. AI Agent Rules Summary

When generating or modifying code in this project, always follow these rules:

1. **Never** expose JPA entities directly from controllers — always map to response DTOs first.
2. **Always** return entities from the service layer — never map to DTOs inside a service.
3. **Always** put mapping logic in the controller layer using the appropriate MapStruct mapper.
4. **Always** use `@Valid` on `@RequestBody` params and query param request objects in controllers.
5. **Always** wrap all API responses in `BaseResponse<T>`.
6. **Always** wrap paginated responses in `BaseResponse<PageResponse<T>>`.
7. **Always** extend `PaginatedRequest` for list requests that support pagination.
8. **All** entities must extend `BaseEntity`. Add `deletedAt`/`deletedBy` manually for soft-delete entities.
9. **Never** use `@Data` on JPA entities — use `@Getter`, `@Setter`, `@NoArgsConstructor` instead.
10. **Always** throw `ValidationException` (not raw `RuntimeException`) for service-layer business rule violations.
11. **Never** handle exceptions inline in controllers — always delegate to `GlobalExceptionHandler`.
12. **Never** hardcode secrets or environment-specific values — use environment variables.
13. **Always** use `@Transactional(readOnly = true)` on read-only service methods.