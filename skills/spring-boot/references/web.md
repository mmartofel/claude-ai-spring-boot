# Web Layer - Controllers & REST APIs

## REST Controller Pattern

```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping
    public PagedModel<UserResponse> listUsers(
            @ParameterObject @PageableDefault(size = 20, sort = "createdAt") Pageable pageable) {
        return new PagedModel<>(userService.findAll(pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse user = userService.findById(id);
        return ResponseEntity.ok(user);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // springdoc documents a ResponseEntity return as 200 without it
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody UserCreateRequest request) {
        UserResponse user = userService.create(request);
        URI location = ServletUriComponentsBuilder
                .fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(user.id())
                .toUri();
        return ResponseEntity.created(location).body(user);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UserUpdateRequest request) {
        UserResponse user = userService.update(id, request);
        return ResponseEntity.ok(user);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## Paged Lists

- Return Spring Data's `PagedModel` (`{content, page: {number, size, totalElements, totalPages}}`), never `Page`/`PageImpl`: their JSON is not a stable format and Spring Data logs a warning when serializing them.
- Annotate `Pageable` with springdoc's `@ParameterObject`, so the contract shows `page`, `size` and `sort` as separate query parameters instead of one `pageable` object.
- Do not put `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)` on the application class to get the same shape: `@WebMvcTest` then registers the web support twice and fails.
- Cap the page size (`spring.data.web.pageable.max-page-size: 100`).
- An unknown `sort` property throws `PropertyReferenceException` (package `org.springframework.data.core` in Spring Data 4); map it to 400 (see Exception Handling).

## Request DTOs with Validation

```java
public record UserCreateRequest(
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$",
             message = "Password must contain uppercase, lowercase, and digit")
    String password,

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username must be alphanumeric")
    String username,

    @Min(value = 18, message = "Must be at least 18")
    @Max(value = 120, message = "Must be at most 120")
    Integer age
) {}

// PUT replaces the whole record: same fields and constraints as the create request,
// plus the version from the last read (409 on mismatch). Optional fields sent as null are cleared.
// Required when the API is built for a UI; a plain API may use another update model, but still sends the version.
public record UserUpdateRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 3, max = 50) @Pattern(regexp = "^[a-zA-Z0-9_]+$") String username,
    @Min(18) @Max(120) Integer age,
    @NotNull Long version
) {}
```

Rules that keep the constraints usable by API clients (the OpenAPI contract carries them as JSON Schema, see `ui-contract.md`). Required when the API is built for a UI (`conventions` skill); recommended otherwise:
- `@Pattern` must also be a valid JavaScript (ECMA-262) regex: no inline flags (`(?s)`, `(?i)`), possessive quantifiers (`a++`), atomic groups (`(?>…)`), `\A`, `\Z`, `\z` or `\p{javaLowerCase}`.
- A required text field is `@NotBlank @Size(min = 1, max = N)`: `@Size(max = N)` alone overwrites the `minLength: 1` that `@NotBlank` produces with `0`.
- A rule Bean Validation cannot express gets a custom constraint, the matching `@Schema` keyword (e.g. `@Schema(multipleOf = 0.5)`) and the same DB `CHECK`.
- A field with a default (e.g. `status`) sent as `null` on PUT gets its default again: the entity setter maps null to it (`Objects.requireNonNullElse`), the request documents it with `@Schema(defaultValue = "ACTIVE")`, and the migration has the same `DEFAULT`.

## Response DTOs

```java
public record UserResponse(
    @Schema(requiredMode = REQUIRED) Long id,
    @Schema(requiredMode = REQUIRED) String email,
    @Schema(requiredMode = REQUIRED) String username,
    Integer age,
    @Schema(requiredMode = REQUIRED) boolean active,
    @Schema(requiredMode = REQUIRED, accessMode = READ_ONLY) Instant createdAt,
    @Schema(requiredMode = REQUIRED, accessMode = READ_ONLY) Instant updatedAt,
    @Schema(requiredMode = REQUIRED) Long version
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getEmail(),
            user.getUsername(),
            user.getAge(),
            user.isActive(),
            user.getCreatedAt(),
            user.getUpdatedAt(),
            user.getVersion()
        );
    }
}
```

When the API is built for a UI, mark every always-present field `requiredMode = REQUIRED`; otherwise clients generated from the contract (TypeScript, Kotlin) see every field as optional. Include `version`, so a client can send it back on PUT.

## Exception Handling (Problem Details)

Every error is an RFC 9457 Problem Details body (`application/problem+json`). When the API is built for a UI (`conventions` skill), field errors go into an `errors` extension, one entry per field with a stable, documented `code` that clients can translate; `message` is an English fallback only. `field` is left out for errors about the whole record.

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Invalid request content.",
  "instance": "/api/users",
  "errors": [
    {"field": "email", "code": "email", "message": "must be a well-formed email address"},
    {"field": "username", "code": "required", "message": "must not be blank"}
  ]
}
```

```java
/** All errors are RFC 9457 Problem Details; field-level errors are listed in {@code errors}. */
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    public static final String REQUIRED = "required";
    public static final String EMAIL = "email";
    public static final String UNIQUE = "unique";
    public static final String INVALID = "invalid";

    /** {@code code} is stable for clients; {@code message} is an English fallback. */
    @Schema(name = "FieldError")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public record FieldMessage(String field,
                               @Schema(requiredMode = Schema.RequiredMode.REQUIRED,
                                       allowableValues = {REQUIRED, EMAIL, UNIQUE, INVALID})
                               String code,
                               @Schema(requiredMode = Schema.RequiredMode.REQUIRED) String message) {
    }

    /** Documents the error body in the OpenAPI contract (see ui-contract.md); keep it in sync with the handler. */
    @Schema(name = "Problem")
    public record Problem(URI type,
                          @Schema(requiredMode = Schema.RequiredMode.REQUIRED) String title,
                          @Schema(requiredMode = Schema.RequiredMode.REQUIRED) int status,
                          String detail, URI instance, List<FieldMessage> errors) {
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers,
                                                                  HttpStatusCode status, WebRequest request) {
        List<FieldMessage> errors = ex.getFieldErrors().stream()
                .map(e -> new FieldMessage(e.getField(), code(e.getCode()), e.getDefaultMessage()))
                .toList();
        // An empty required text breaks both @NotBlank and @Size(min = 1); report only "required" for that field.
        Set<String> missing = errors.stream().filter(e -> REQUIRED.equals(e.code())).map(FieldMessage::field)
                .collect(Collectors.toSet());
        ex.getBody().setProperty("errors", errors.stream()
                .filter(e -> REQUIRED.equals(e.code()) || !missing.contains(e.field()))
                .toList());
        return super.handleMethodArgumentNotValid(ex, headers, status, request);
    }

    /** Maps a Bean Validation constraint name ({@code FieldError#getCode()}) to a stable error code. */
    private static String code(String constraint) {
        return switch (constraint) {
            case "NotBlank", "NotNull" -> REQUIRED;
            case "Email" -> EMAIL;
            case null, default -> INVALID;
        };
    }

    /** 409 with a field error for a known unique constraint, 409 without {@code errors} otherwise. */
    @ExceptionHandler
    ProblemDetail handleDataIntegrity(DataIntegrityViolationException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        if (ex.getCause() instanceof ConstraintViolationException cve
                && User.UNIQUE_EMAIL.equals(cve.getConstraintName())) {
            problem.setDetail("Email is already in use");
            problem.setProperty("errors", List.of(new FieldMessage("email", UNIQUE, "already in use")));
        }
        return problem;
    }

    /** 409 without {@code errors}: the client sent a stale version and has to reload. */
    @ExceptionHandler
    ProblemDetail handleOptimisticLock(ObjectOptimisticLockingFailureException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, "Record was changed by someone else; reload and retry");
    }

    /** 400 for {@code ?sort=unknownProperty}. */
    @ExceptionHandler
    ProblemDetail handleBadSort(PropertyReferenceException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    /** 500 for anything not handled above; the cause is logged, never returned to the client. */
    @ExceptionHandler
    ProblemDetail handleUnexpected(Exception ex) {
        logger.error("Unexpected error", ex);
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Unexpected server error");
    }
}
```

- Extend `ResponseEntityExceptionHandler`: it already renders Spring MVC's exceptions (`MethodArgumentNotValidException`, `ResponseStatusException`, `ErrorResponseException`, ...) as Problem Details.
- `ConstraintViolationException` here is Hibernate's (`org.hibernate.exception`), which carries the database constraint name; name constraints explicitly in migrations (`uk_users_email`) and keep the name in a constant on the entity.
- A new constraint or code goes into `code(...)`, the `allowableValues` of `FieldError`, the README and the tests at the same time. Clients translate codes, never `message`.
- The catch-all has no `@ResponseStatus`, so springdoc does not add a 500 response to every operation.
- Never put `ex.getMessage()` of an unexpected exception into the response.
- A missing record throws `ResponseStatusException(NOT_FOUND, ...)` or a prepared `ErrorResponseException`; `ResponseEntityExceptionHandler` renders both, no extra `@ExceptionHandler` needed.
- Deleting a record that other records still reference returns 409 with a `detail`; it never cascades (the foreign key is `ON DELETE RESTRICT` as a backstop).

## Cross-Field Validation

A rule that involves two fields is a class-level constraint that reports the error on one field (the later one), so a client can show it next to an input:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRange.Validator.class)
public @interface DateRange {
    String message() default "must not be before the start date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

    class Validator implements ConstraintValidator<DateRange, DatePeriod> {
        @Override
        public boolean isValid(DatePeriod value, ConstraintValidatorContext context) {
            if (value.from() == null || value.to() == null || !value.to().isBefore(value.from())) {
                return true;
            }
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                    .addPropertyNode("to").addConstraintViolation();
            return false;
        }
    }
}

/** Implemented by the create and update requests that carry the period. */
public interface DatePeriod {
    LocalDate from();
    LocalDate to();
}
```

Map `DateRange` to its own code (e.g. `date-range`) in the exception handler.

## Custom Validation

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;
        return !userRepository.existsByEmail(email);
    }
}
```

A lookup like this gives an early, friendly error but races with concurrent inserts and rejects a PUT that keeps its own email. The database unique constraint stays the source of truth, mapped to 409 `unique` in `ApiExceptionHandler`.

## RestClient for External APIs

```java
@Configuration
public class RestClientConfig {
    private static final Logger log = LoggerFactory.getLogger(RestClientConfig.class);

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .requestInterceptor((request, body, execution) -> {
                log.info("Request: {} {}", request.getMethod(), request.getURI());
                return execution.execute(request, body);
            })
            .build();
    }
}

@Service
public class ExternalApiService {
    private final RestClient restClient;

    public ExternalApiService(RestClient restClient) {
        this.restClient = restClient;
    }

    public ExternalDataResponse fetchData(String id) {
        return restClient
            .get()
            .uri("/data/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new ResourceNotFoundException("External resource not found");
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new ServiceUnavailableException("External service unavailable");
            })
            .body(ExternalDataResponse.class);
    }
}
```

Use `RestClient` for blocking calls (default with virtual threads). Use `WebClient` only for reactive/streaming scenarios.

## CORS Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

## Problem Details (RFC 9457)

```yaml
# application.yml - enables standard error response format
spring:
  mvc:
    problemdetails:
      enabled: true
```

With this enabled, Spring Boot returns RFC 9457 responses (which replaced RFC 7807) for standard exceptions, also without a custom `@RestControllerAdvice`:

```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "User with id 42 not found",
  "instance": "/api/v1/users/42"
}
```

Custom Problem Detail exceptions:

```java
public class UserNotFoundException extends ErrorResponseException {

    public UserNotFoundException(Long userId) {
        super(HttpStatus.NOT_FOUND,
            ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND,
                "User with id " + userId + " not found"),
            null);
        setTitle("User Not Found");
        setType(URI.create("https://api.example.com/errors/user-not-found"));
    }
}
```

## Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@RestController` | Marks class as REST controller (combines @Controller + @ResponseBody) |
| `@RequestMapping` | Maps HTTP requests to handler methods |
| `@GetMapping/@PostMapping` | HTTP method-specific mappings |
| `@PathVariable` | Extracts values from URI path |
| `@RequestParam` | Extracts query parameters |
| `@RequestBody` | Binds request body to method parameter |
| `@Valid` | Triggers validation on request body |
| `@RestControllerAdvice` | Global exception handling for REST controllers |
| `@ResponseStatus` | Sets HTTP status code for method |
| `@ParameterObject` | (springdoc) Documents `Pageable` or a filter record as separate query parameters |
| `PagedModel` | Stable JSON for a page: `{content, page}` |
