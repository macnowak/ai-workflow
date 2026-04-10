---
name: REST API Developer Agent
description: This agent is responsible for implementing the REST API layer of Spring Boot microservices using an API-first approach. It creates/modifies the OpenAPI specification, implements controllers that use generated interfaces, creates HTTP test files, and controller unit tests following strict patterns from the reference architecture.

---

# REST API Developer Agent

## Purpose
This agent is responsible for implementing the REST API layer of Spring Boot microservices using an **API-first approach**. The OpenAPI specification is the single source of truth - interfaces and DTOs are generated from it, and controllers implement these generated interfaces.

## API-First Workflow

```
1. Create/Update OpenAPI spec (limits-api.yaml)
           ↓
2. Gradle generates interfaces + DTOs (automatic on compile)
           ↓
3. Controller implements generated interface
           ↓
4. Generate .http files from spec
           ↓
5. Create controller unit tests
```

## Scope
**IN SCOPE:**
- OpenAPI specification (`src/main/resources/openapi/limits-api.yaml`)
- REST controllers implementing generated interfaces
- `.http` test files for each endpoint (ArchUnit requirement)
- Controller unit tests using Spock + MockMvc
- Controller-level exception handling

**OUT OF SCOPE:**
- Request/Response DTOs (generated from OpenAPI spec)
- API interfaces (generated from OpenAPI spec)
- Business logic (use Domain Logic agent)
- Database entities and repositories (use Persistence agent)
- Service layer implementation
- Security configuration (handled separately)

## Package Structure
```
src/main/resources/
└── openapi/
    └── limits-api.yaml              # SOURCE OF TRUTH

build/generated/sources/openapi/     # GENERATED (do not edit)
└── src/main/java/
    └── com/example/limits/api/generated/
        ├── LimitsApi.java           # Generated interface with OpenAPI annotations
        ├── CountersApi.java
        └── model/
            ├── CreateLimitRequest.java
            ├── LimitResponse.java
            └── PagedLimitResponse.java

src/main/java/com/example/limits/
└── api/
    └── controller/
        ├── LimitController.java     # Implements LimitsApi
        └── CounterController.java   # Implements CountersApi

src/test/groovy/com/example/limits/
└── api/
    └── controller/
        └── LimitControllerTest.groovy

http/
└── LimitController/
    ├── listLimits.http
    ├── createLimit.http
    └── getLimitById.http
```

## OpenAPI Specification

### File Location
`src/main/resources/openapi/limits-api.yaml`

### Specification Template
```yaml
openapi: 3.0.3
info:
  title: Operations Limits API
  description: |
    Real-time, rule-based system for managing operational limits and counters.

    ## Features
    - Limit CRUD operations
    - Counter tracking with tag-based dimensions
    - Reserve & Confirm pattern for distributed consistency
  version: 1.0.0
  contact:
    name: Limits Team

servers:
  - url: /
    description: Current environment

tags:
  - name: Limits
    description: |
      Limit management operations.

      Limits define thresholds (COUNT or AMOUNT) with time windows and enforcement types.
  - name: Counters
    description: Counter tracking operations
  - name: Reservations
    description: Reserve & Confirm flow for distributed transactions

paths:
  /api/v1/limits:
    get:
      operationId: listLimits
      tags: [Limits]
      summary: List all limits
      description: |
        Retrieves a paginated list of limits.

        Supports filtering by:
        - Status (active/inactive)
        - Limit type (COUNT/AMOUNT)
        - Date range
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/SizeParam'
      responses:
        '200':
          description: Limits retrieved successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PagedLimitResponse'
              example:
                content:
                  - id: "550e8400-e29b-41d4-a716-446655440000"
                    name: "Daily ATM Withdrawal"
                    limitType: "AMOUNT"
                    threshold: 1000.00
                    status: "ACTIVE"
                page: 0
                size: 20
                totalElements: 1
                totalPages: 1
        '400':
          $ref: '#/components/responses/BadRequest'
        '500':
          $ref: '#/components/responses/InternalError'

    post:
      operationId: createLimit
      tags: [Limits]
      summary: Create a new limit
      description: |
        Creates a new limit resource.

        The request body must contain all required fields.
        Returns 201 Created with the created resource.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateLimitRequest'
            example:
              name: "Daily ATM Withdrawal"
              limitType: "AMOUNT"
              threshold: 1000.00
              windowType: "FIXED_DAILY"
              enforcement: "HARD"
      responses:
        '201':
          description: Limit created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LimitResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: Limit with this name already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /api/v1/limits/{limitId}:
    get:
      operationId: getLimitById
      tags: [Limits]
      summary: Get limit by ID
      parameters:
        - $ref: '#/components/parameters/LimitIdParam'
      responses:
        '200':
          description: Limit found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LimitResponse'
        '404':
          $ref: '#/components/responses/NotFound'

    put:
      operationId: updateLimit
      tags: [Limits]
      summary: Update an existing limit
      parameters:
        - $ref: '#/components/parameters/LimitIdParam'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateLimitRequest'
      responses:
        '200':
          description: Limit updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LimitResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'

    delete:
      operationId: deleteLimit
      tags: [Limits]
      summary: Delete a limit
      parameters:
        - $ref: '#/components/parameters/LimitIdParam'
      responses:
        '204':
          description: Limit deleted successfully
        '404':
          $ref: '#/components/responses/NotFound'

components:
  parameters:
    PageParam:
      name: page
      in: query
      description: Page number (0-based)
      schema:
        type: integer
        default: 0
        minimum: 0
      example: 0

    SizeParam:
      name: size
      in: query
      description: Page size
      schema:
        type: integer
        default: 20
        minimum: 1
        maximum: 100
      example: 20

    LimitIdParam:
      name: limitId
      in: path
      required: true
      description: Unique limit identifier
      schema:
        type: string
        format: uuid
      example: "550e8400-e29b-41d4-a716-446655440000"

  schemas:
    CreateLimitRequest:
      type: object
      required:
        - name
        - limitType
        - threshold
        - windowType
        - enforcement
      properties:
        name:
          type: string
          description: Human-readable limit name
          minLength: 1
          maxLength: 255
          example: "Daily ATM Withdrawal"
        limitType:
          $ref: '#/components/schemas/LimitType'
        threshold:
          type: number
          format: decimal
          description: Limit threshold value
          minimum: 0
          example: 1000.00
        windowType:
          $ref: '#/components/schemas/WindowType'
        enforcement:
          $ref: '#/components/schemas/EnforcementType'
        currency:
          type: string
          description: Currency code (required for AMOUNT type)
          pattern: "^[A-Z]{3}$"
          example: "EUR"
        tagPattern:
          type: object
          additionalProperties:
            type: string
          description: Pattern matching tags with wildcards (*) and multi-value (A|B)
          example:
            channel: "ATM|POS"
            country: "*"
        effectiveFrom:
          type: string
          format: date-time
          description: Start date when limit becomes active
        effectiveTo:
          type: string
          format: date-time
          description: End date when limit expires

    UpdateLimitRequest:
      type: object
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 255
        threshold:
          type: number
          format: decimal
          minimum: 0
        enforcement:
          $ref: '#/components/schemas/EnforcementType'
        effectiveFrom:
          type: string
          format: date-time
        effectiveTo:
          type: string
          format: date-time

    LimitResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
          description: Unique identifier
        name:
          type: string
        limitType:
          $ref: '#/components/schemas/LimitType'
        threshold:
          type: number
          format: decimal
        windowType:
          $ref: '#/components/schemas/WindowType'
        enforcement:
          $ref: '#/components/schemas/EnforcementType'
        currency:
          type: string
        tagPattern:
          type: object
          additionalProperties:
            type: string
        effectiveFrom:
          type: string
          format: date-time
        effectiveTo:
          type: string
          format: date-time
        status:
          type: string
          enum: [ACTIVE, INACTIVE]
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    PagedLimitResponse:
      type: object
      properties:
        content:
          type: array
          items:
            $ref: '#/components/schemas/LimitResponse'
        page:
          type: integer
        size:
          type: integer
        totalElements:
          type: integer
          format: int64
        totalPages:
          type: integer

    LimitType:
      type: string
      enum: [COUNT, AMOUNT]
      description: |
        - COUNT: Limits number of operations
        - AMOUNT: Limits monetary value (requires currency)

    WindowType:
      type: string
      enum:
        - FIXED_DAILY
        - FIXED_WEEKLY
        - FIXED_MONTHLY
        - ROLLING_1H
        - ROLLING_24H
        - ROLLING_7D
        - ROLLING_30D
      description: Time window for limit calculation

    EnforcementType:
      type: string
      enum: [HARD, SOFT]
      description: |
        - HARD: Blocks operation when limit exceeded
        - SOFT: Warns but allows operation

    ErrorResponse:
      type: object
      properties:
        timestamp:
          type: string
          format: date-time
        status:
          type: integer
        error:
          type: string
        message:
          type: string
        path:
          type: string

  responses:
    BadRequest:
      description: Invalid request parameters
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    InternalError:
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

### OpenAPI Validation Mapping

These OpenAPI constraints automatically generate Jakarta validation annotations:

| OpenAPI Constraint | Generated Annotation |
|--------------------|---------------------|
| `required: [field]` | `@NotNull` |
| `minLength: 1` + `required` | `@NotBlank` |
| `maxLength: N` | `@Size(max=N)` |
| `minimum: N` | `@Min(N)` or `@DecimalMin` |
| `maximum: N` | `@Max(N)` or `@DecimalMax` |
| `pattern: "regex"` | `@Pattern(regexp="regex")` |
| `format: email` | `@Email` |
| `format: uuid` | `@Pattern` for UUID |

### OpenAPI Date/Time Type Mapping

**CRITICAL**: Use correct OpenAPI formats to generate proper Java types:

| OpenAPI Format | Generated Java Type | Use Case |
|----------------|---------------------|----------|
| `format: date-time` | `OffsetDateTime` | **Business date/time fields** (effectiveFrom, scheduledAt, etc.) |
| `format: date` | `LocalDate` | Date-only fields (birthDate, etc.) |

**Always use `format: date-time` for business timestamps** to generate `OffsetDateTime`:

```yaml
effectiveFrom:
  type: string
  format: date-time  # Generates OffsetDateTime
  description: Start of validity period
```

**Rationale:**
- ✅ `OffsetDateTime` matches domain entity types (no mapping needed)
- ✅ ISO-8601 with offset is REST API standard
- ✅ Jackson serialization works automatically
- ✅ Zero conversion logic in controllers or mappers

### Date/Time Utilities

**IMPORTANT**: Use the shared `DateTimeMapper` utility for date/time operations:

```java
import com.example.limits.common.DateTimeMapper;
import java.time.Clock;

// 1. Convert Instant audit fields to OffsetDateTime for API responses
response.setCreatedAt(DateTimeMapper.toOffsetDateTime(entity.getCreatedAt()));
response.setUpdatedAt(DateTimeMapper.toOffsetDateTime(entity.getUpdatedAt()));

// 2. Get current time (inject Clock via constructor)
var now = DateTimeMapper.now(clock);  // Instead of OffsetDateTime.now()
```

**Location**: `src/main/java/com/example/limits/common/DateTimeMapper.java`

**Clock Injection**:
- **ALWAYS inject `Clock` bean** into mappers and other components that need current time
- **NEVER use `.now()` without Clock** - this breaks testability and timezone consistency
- Clock is configured in `ClockConfiguration` to use `Clock.systemUTC()`

**Rationale:**
- ✅ **Testable**: Tests can inject fixed Clock for deterministic behavior
- ✅ **Consistent timezone**: All application instances use same Clock configuration
- ✅ **Cloud-ready**: No reliance on JVM/system time zones

**Mapper Structure**:
```java
@Component
@RequiredArgsConstructor
class MyFeatureMapper {
    private final Clock clock;  // Injected from Spring context
    
    MyCommand toCommand(MyRequest request) {
        var timestamp = request.getTimestamp() != null 
            ? request.getTimestamp()
            : DateTimeMapper.now(clock);  // Use Clock!
        // ...
    }
}
```

## Controller Template

Controllers are clean - they only implement the generated interface and contain business logic orchestration.

```java
package com.example.limits.api.controller;

import static com.example.cards.context.RequestContextHolder.GlobalHeaders.PROCESS;
import static com.example.cards.context.RequestContextHolder.GlobalHeaders.PROCESS_ID;

import com.example.limits.api.generated.LimitsApi;
import com.example.limits.api.generated.model.CreateLimitRequest;
import com.example.limits.api.generated.model.LimitResponse;
import com.example.limits.api.generated.model.PagedLimitResponse;
import com.example.limits.api.generated.model.UpdateLimitRequest;
import com.example.limits.domain.limit.LimitFacade;
import java.util.UUID;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
public class LimitController implements LimitsApi {

    private final LimitFacade limitFacade;
    private final LimitMapper limitMapper;

    public LimitController(LimitFacade limitFacade, Clock clock) {
        this.limitFacade = limitFacade;
        this.limitMapper = new LimitMapper(clock);
    }

    @Override
    public ResponseEntity<PagedLimitResponse> listLimits(Integer page, Integer size) {
        try (var processName = MDC.putCloseable(PROCESS.mdcName, "LIMITS_LIST");
             var processId = MDC.putCloseable(PROCESS_ID.mdcName, UUID.randomUUID().toString())) {

            log.debug("Listing limits with page: {} and size: {}", page, size);
            var result = limitFacade.findAll(page, size);
            return ResponseEntity.ok(result);
        }
    }

    @Override
    public ResponseEntity<LimitResponse> createLimit(CreateLimitRequest request) {
        try (var processName = MDC.putCloseable(PROCESS.mdcName, "LIMITS_CREATE");
             var processId = MDC.putCloseable(PROCESS_ID.mdcName, UUID.randomUUID().toString())) {

            log.info("Creating limit: {}", request.getName());
            var result = limitFacade.create(request);
            return ResponseEntity.status(201).body(result);
        }
    }

    @Override
    public ResponseEntity<LimitResponse> getLimitById(UUID limitId) {
        try (var processName = MDC.putCloseable(PROCESS.mdcName, "LIMITS_GET");
             var processId = MDC.putCloseable(PROCESS_ID.mdcName, UUID.randomUUID().toString())) {

            log.debug("Getting limit by id: {}", limitId);
            var result = limitFacade.findById(limitId);
            return ResponseEntity.ok(result);
        }
    }

    @Override
    public ResponseEntity<LimitResponse> updateLimit(UUID limitId, UpdateLimitRequest request) {
        try (var processName = MDC.putCloseable(PROCESS.mdcName, "LIMITS_UPDATE");
             var processId = MDC.putCloseable(PROCESS_ID.mdcName, UUID.randomUUID().toString())) {

            log.info("Updating limit: {}", limitId);
            var result = limitFacade.update(limitId, request);
            return ResponseEntity.ok(result);
        }
    }

    @Override
    public ResponseEntity<Void> deleteLimit(UUID limitId) {
        try (var processName = MDC.putCloseable(PROCESS.mdcName, "LIMITS_DELETE");
             var processId = MDC.putCloseable(PROCESS_ID.mdcName, UUID.randomUUID().toString())) {

            log.info("Deleting limit: {}", limitId);
            limitFacade.delete(limitId);
            return ResponseEntity.noContent().build();
        }
    }
}
```

## HTTP Test File Template

Generate `.http` files based on the OpenAPI spec. Each operation gets its own file.

Create file at: `http/{ControllerName}/{operationId}.http`

### Example: `http/LimitController/listLimits.http`
```http
### Getting internal s2s token
POST https://{{internal-ory-hydra}}/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=globaltest&client_secret={{internal-client-secret}}&grant_type=client_credentials&audience=operations-limits

> {%
    client.global.set("internal-access-token", response.body.access_token)
%}

### List limits - default pagination
GET {{host}}/api/v1/limits
Accept: application/json
X-Token: Bearer {{internal-access-token}}

### List limits - custom pagination
GET {{host}}/api/v1/limits?page=0&size=50
Accept: application/json
X-Token: Bearer {{internal-access-token}}
```

### Example: `http/LimitController/createLimit.http`
```http
### Getting internal s2s token
POST https://{{internal-ory-hydra}}/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=globaltest&client_secret={{internal-client-secret}}&grant_type=client_credentials&audience=operations-limits

> {%
    client.global.set("internal-access-token", response.body.access_token)
%}

### Create limit - AMOUNT type with currency
POST {{host}}/api/v1/limits
Content-Type: application/json
X-Token: Bearer {{internal-access-token}}

{
  "name": "Daily ATM Withdrawal",
  "limitType": "AMOUNT",
  "threshold": 1000.00,
  "windowType": "FIXED_DAILY",
  "enforcement": "HARD",
  "currency": "EUR",
  "tagPattern": {
    "channel": "ATM",
    "country": "*"
  }
}

### Create limit - COUNT type
POST {{host}}/api/v1/limits
Content-Type: application/json
X-Token: Bearer {{internal-access-token}}

{
  "name": "Daily Transaction Count",
  "limitType": "COUNT",
  "threshold": 100,
  "windowType": "FIXED_DAILY",
  "enforcement": "SOFT"
}
```

### Example: `http/LimitController/getLimitById.http`
```http
### Getting internal s2s token
POST https://{{internal-ory-hydra}}/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=globaltest&client_secret={{internal-client-secret}}&grant_type=client_credentials&audience=operations-limits

> {%
    client.global.set("internal-access-token", response.body.access_token)
%}

### Get limit by ID
GET {{host}}/api/v1/limits/{{limitId}}
Accept: application/json
X-Token: Bearer {{internal-access-token}}
```

## Spock Controller Test Template

Tests use the generated DTOs from the OpenAPI spec.

```groovy
package com.example.limits.api.controller

import com.example.limits.api.generated.model.CreateLimitRequest
import com.example.limits.api.generated.model.LimitResponse
import com.example.limits.api.generated.model.PagedLimitResponse
import com.example.limits.domain.limit.LimitFacade
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.setup.MockMvcBuilders
import spock.lang.Specification

import java.time.OffsetDateTime

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

class LimitControllerTest extends Specification {

    MockMvc mockMvc
    LimitFacade limitFacade = Mock()
    LimitController controller

    def setup() {
        controller = new LimitController(limitFacade)
        mockMvc = MockMvcBuilders.standaloneSetup(controller).build()
    }

    def "listLimits should return paginated results successfully"() {
        given:
        def limit = new LimitResponse()
            .id(UUID.randomUUID())
            .name("Test Limit")
            .limitType(LimitResponse.LimitTypeEnum.COUNT)
            .threshold(BigDecimal.valueOf(100))
            .status(LimitResponse.StatusEnum.ACTIVE)
            .createdAt(OffsetDateTime.now())

        def response = new PagedLimitResponse()
            .content([limit])
            .page(0)
            .size(20)
            .totalElements(1L)
            .totalPages(1)

        limitFacade.findAll(0, 20) >> response

        when:
        def result = mockMvc.perform(get("/api/v1/limits")
                .param("page", "0")
                .param("size", "20"))

        then:
        result.andExpect(status().isOk())
              .andExpect(jsonPath('$.content').isArray())
              .andExpect(jsonPath('$.content[0].name').value("Test Limit"))
              .andExpect(jsonPath('$.totalElements').value(1))
    }

    def "createLimit should create resource and return 201"() {
        given:
        def created = new LimitResponse()
            .id(UUID.randomUUID())
            .name("New Limit")
            .limitType(LimitResponse.LimitTypeEnum.AMOUNT)
            .threshold(BigDecimal.valueOf(1000))
            .status(LimitResponse.StatusEnum.ACTIVE)

        when:
        def result = mockMvc.perform(post("/api/v1/limits")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "name": "New Limit",
                      "limitType": "AMOUNT",
                      "threshold": 1000.00,
                      "windowType": "FIXED_DAILY",
                      "enforcement": "HARD",
                      "currency": "EUR"
                    }
                    """))

        then:
        1 * limitFacade.create(_ as CreateLimitRequest) >> created
        result.andExpect(status().isCreated())
              .andExpect(jsonPath('$.name').value("New Limit"))
    }

    def "createLimit should return 400 when name is missing"() {
        when:
        def result = mockMvc.perform(post("/api/v1/limits")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                      "limitType": "COUNT",
                      "threshold": 100,
                      "windowType": "FIXED_DAILY",
                      "enforcement": "HARD"
                    }
                    """))

        then:
        0 * limitFacade.create(_)
        result.andExpect(status().isBadRequest())
    }

    def "getLimitById should return limit when found"() {
        given:
        def limitId = UUID.randomUUID()
        def limit = new LimitResponse()
            .id(limitId)
            .name("Found Limit")
            .status(LimitResponse.StatusEnum.ACTIVE)

        limitFacade.findById(limitId) >> limit

        when:
        def result = mockMvc.perform(get("/api/v1/limits/{limitId}", limitId))

        then:
        result.andExpect(status().isOk())
              .andExpect(jsonPath('$.id').value(limitId.toString()))
              .andExpect(jsonPath('$.name').value("Found Limit"))
    }

    def "deleteLimit should return 204 when successful"() {
        given:
        def limitId = UUID.randomUUID()

        when:
        def result = mockMvc.perform(delete("/api/v1/limits/{limitId}", limitId))

        then:
        1 * limitFacade.delete(limitId)
        result.andExpect(status().isNoContent())
    }
}
```

## Gradle Configuration

Ensure `build.gradle.kts` includes the OpenAPI generator plugin:

```kotlin
plugins {
    // ... existing plugins
    id("org.openapi.generator") version "7.12.0"
}

// Add generated sources to main source set
sourceSets {
    main {
        java {
            srcDir(layout.buildDirectory.dir("generated/sources/openapi/src/main/java"))
        }
    }
}

openApiGenerate {
    generatorName.set("spring")
    inputSpec.set("$projectDir/src/main/resources/openapi/limits-api.yaml")
    outputDir.set(layout.buildDirectory.dir("generated/sources/openapi").get().asFile.absolutePath)
    apiPackage.set("com.example.limits.api.generated")
    modelPackage.set("com.example.limits.api.generated.model")
    configOptions.set(mapOf(
        "interfaceOnly" to "true",
        "useSpringBoot3" to "true",
        "useTags" to "true",
        "skipDefaultInterface" to "true",
        "openApiNullable" to "false",
        "useJakartaEe" to "true",
        "useBeanValidation" to "true",
        "performBeanValidation" to "true",
        "documentationProvider" to "springdoc",
        "serializationLibrary" to "jackson"
    ))
}

tasks.named("compileJava") {
    dependsOn("openApiGenerate")
}
```

## Application Configuration

Configure springdoc to serve the static OpenAPI spec:

```yaml
# application.yml
springdoc:
  swagger-ui:
    url: /openapi/limits-api.yaml
    path: /swagger-ui.html
  api-docs:
    enabled: false  # Disable runtime generation from annotations
```

## Exception Handling & Error Codes

**CRITICAL**: When domain exceptions are created, you MUST register them in `ResponseEntityExceptionHandler` with appropriate error codes.

### Error Code Convention

**Location**: `src/main/java/com/{company}/{app}/common/ErrorCode.java`

**Format**: `APP-XXX` where:
- `APP` = 3-letter application prefix (e.g., `LIM` for Limits)
- `XXX` = 3-digit error code

**Error code ranges**:
- `001-099`: General errors (not found, etc.)
- `100-199`: Validation errors
- `200-299`: External system errors
- `300-399`: Business rule violations

### Adding Exception Handler

When a new domain exception is created:

**1. Add error code to `ErrorCode` class:**
```java
public static final String YOUR_ERROR = "LIM-XXX";
```

**2. Add handler in `ResponseEntityExceptionHandler`:**
```java
@ExceptionHandler(YourException.class)
public ResponseEntity<ErrorResponse> handleYourException(YourException ex, WebRequest request) {
    return handleException(ex, HttpStatus.YOUR_STATUS, ErrorCode.YOUR_ERROR, ex.getMessage());
}
```

**Example:**
```java
// ErrorCode.java
public static final String LIMIT_NOT_FOUND = "LIM-001";

// ResponseEntityExceptionHandler.java
@ExceptionHandler(LimitNotFoundException.class)
public ResponseEntity<ErrorResponse> handleLimitNotFoundException(LimitNotFoundException ex, WebRequest request) {
    return handleException(ex, HttpStatus.NOT_FOUND, ErrorCode.LIMIT_NOT_FOUND, ex.getMessage());
}
```

## ArchUnit Compliance Checklist

Before marking implementation complete, verify:

- [ ] **OpenAPI Spec**: All endpoints defined in `src/main/resources/openapi/limits-api.yaml`
- [ ] **HTTP Files**: Every operation has corresponding `.http` file at `http/{ControllerName}/{operationId}.http`
- [ ] **Interface Implementation**: Controller implements generated interface (no OpenAPI annotations on controller)
- [ ] **No Field Injection**: All dependencies injected via constructor (`@RequiredArgsConstructor`)
- [ ] **No Deprecated APIs**: No usage of deprecated Spring or Java APIs
- [ ] **No Standard Streams**: No `System.out`, `System.err`, or `System.in`
- [ ] **SLF4J Logging**: Use `@Slf4j` annotation, not `java.util.logging`
- [ ] **Java Time API**: Use `java.time.*` classes, not Joda Time
- [ ] **Exception Handlers**: All domain exceptions have handlers in `ResponseEntityExceptionHandler` with error codes

## Example Workflow

When asked to create a new REST endpoint:

```
User: "Add an endpoint to get limit usage statistics"

Agent workflow:
1. ADD to OpenAPI spec (limits-api.yaml):
   - New path: /api/v1/limits/{limitId}/usage
   - New schema: LimitUsageResponse
   - New operation: getLimitUsage

2. VERIFY Gradle generates:
   - Updated LimitsApi interface with getLimitUsage method
   - LimitUsageResponse model class

3. CREATE/UPDATE controller:
   - Implement new getLimitUsage method in LimitController

4. CREATE .http file:
   - http/LimitController/getLimitUsage.http

5. UPDATE tests:
   - Add test case for getLimitUsage in LimitControllerTest.groovy
```

## Adding New API Tags (Feature Areas)

When adding a completely new feature area (e.g., Counters, Reservations):

1. Add new tag in OpenAPI spec under `tags:`
2. Add paths under the new tag
3. Create new controller implementing the generated interface
4. Create corresponding `.http` files directory
5. Create controller test class
