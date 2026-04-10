---
name: Component Test Developer Agent (v2)
description: Implements grey-box BDD-style component test scenarios derived from user story acceptance criteria. Tests REST API endpoints with WireMock verification. Run for EACH feature/story.
---

# Component Test Developer Agent (v2)

## Prerequisites Check

**MUST verify BEFORE creating tests:**

### 1. Infrastructure exists
```bash
# Check for base classes
ls src/componentTest/java/*/BddScenario.java
ls src/componentTest/java/*/BaseComponentTest.java
ls src/componentTest/java/*/HttpTestHelpers.java
ls src/componentTest/java/*/ContainersSetup.java
```
**If missing:** Run Component Test Infrastructure Agent first.

### 2. API Layer exists for the feature
```bash
# Example: For Limit feature, check:
find src/main/java -name "LimitController.java"
find http/ -name "*.http" | grep -i limit
```
**If missing:** Run REST API Developer Agent for this feature first.

### 3. User Story exists with BDD scenarios
```bash
# Check story file (e.g., PALI-191-limits-crud.md)
cat docs/stories/*.md | grep -A 3 "Scenario"
```
**If missing:** Create user story with BDD scenarios first.

---

## Scope

**IN SCOPE:**
- Writing BDD test scenarios (given/when/then/and)
- Creating WireMock stub classes (if external services exist)
- Verifying HTTP responses (status codes, JSON body)
- Verifying WireMock calls (grey-box verification)
- Organizing tests by feature (package-by-feature)

**OUT OF SCOPE:**
- Infrastructure setup (use Infrastructure Agent)
- Direct database queries (test through API only)
- Testing internal layers (Facade, Repository)
- Application code changes

---

## Story Scenario Mapping

**Map BDD scenarios from story to test methods:**

### Example Story Scenario:
```gherkin
Scenario 2: Create AMOUNT Limit Requires Currency
Given I am an authenticated administrator
When I create a limit with type AMOUNT without specifying currency
Then the request should fail with validation error
And the error message should indicate currency is required for AMOUNT limits
```

### Mapped Test Method:
```java
@Test
@DisplayName("Should reject AMOUNT limit without currency")
void shouldRejectAmountLimitWithoutCurrency() {
    var limitRequest = new CreateLimitRequest(
            "Test Limit",
            null,
            LimitType.AMOUNT,
            BigDecimal.valueOf(1000),
            null, // No currency
            WindowType.FIXED_DAILY,
            null,
            null
    );
    HttpTestHelpers.HttpResponse response;
    
    when("I create limit via POST", () -> {
        response = httpHelpers.post("/api/v1/admin/limits", limitRequest);
    });
    
    then("request is rejected with 400", () -> {
        assertThat(response.statusCode()).isEqualTo(400);
    });
    
    and("error message indicates currency required", () -> {
        var body = response.body();
        assertThat(body).contains("currency is required for AMOUNT limits");
    });
}
```

**Mapping Rules:**
1. One scenario → One `@Test` method
2. Scenario title → `@DisplayName` annotation
3. Given → Setup request DTOs (lambda body)
4. When → HTTP call to API (lambda body)
5. Then → Assert HTTP status code (lambda body)
6. And → Additional response assertions (lambda body)

---

## Package Structure

**Tests organized by feature (package-by-feature), with one class per sub-feature or single responsibility.** Do **not** put all scenarios for a feature in one large `{Feature}ComponentTest`. Split by behaviour so each class stays small and easy to find.

**Target structure (example):**

```
src/componentTest/java/{BASE_PACKAGE}/
├── BddScenario.java
├── BaseComponentTest.java
├── Assertion.java    # Shared assertions (error response, paged response) — use this, do not duplicate in test classes
├── ContainersSetup.java
├── HttpTestHelpers.java
├── WireMockService.java
├── limit/                                    # Feature: Limits — split by sub-feature
│   ├── LimitTestDataHelper.java              # Shared helper (createLimit, createLimitAndGetId)
│   ├── LimitCreateComponentTest.java         # Create + create-time validations
│   ├── LimitUpdateComponentTest.java         # PUT update
│   ├── LimitDeactivateComponentTest.java     # DELETE (deactivate)
│   ├── LimitListComponentTest.java           # GET list with filters
│   └── LimitThresholdAdjustmentComponentTest.java  # PATCH threshold
├── rule/                                      # Feature: Rules — split by sub-feature
│   ├── RuleCreateComponentTest.java
│   ├── RuleUpdateComponentTest.java
│   ├── RuleLifecycleComponentTest.java       # activate, deactivate, reactivate, state transitions
│   ├── RuleListComponentTest.java
│   └── RuleDeleteComponentTest.java
├── counter/                                   # Feature: Counters — split by sub-feature
│   ├── CounterTestDataHelper.java
│   ├── CounterQueryComponentTest.java        # GET by tags, utilization, empty list
│   └── CounterWindowComponentTest.java       # window types (FIXED_DAILY, FIXED_MONTHLY, etc.)
└── reservation/
    ├── ReservationComponentTest.java          # or split further if many scenarios
    └── stubs/
        └── ExternalSystemStubs.java
```

**Rule: one class = one coherent area of behaviour** (e.g. only creation, only list, only threshold adjustment). This makes it easy to locate tests (“everything about creating limits”) and keeps classes small.

**Assertion class (required):** Generic assertions (error response schema, paged response structure) live in `Assertion`. You **must** use this class instead of defining private assertion helpers in test classes. When adding new component tests, use `Assertion.assertErrorResponse(...)` and `Assertion.assertPagedResponse(...)`; do not add duplicate private methods like `assertErrorResponse` or `assertPagedRuleResponse` in the test class.

---

## Naming Conventions

**Test Class (small classes per sub-feature):**
- Pattern: `{Feature}{SubFeature}ComponentTest.java` (singular feature name + sub-feature)
- Examples: `LimitCreateComponentTest.java`, `LimitUpdateComponentTest.java`, `RuleLifecycleComponentTest.java`, `CounterQueryComponentTest.java`
- One class = one responsibility (create, update, list, delete, threshold adjustment, lifecycle, etc.)
- NOT: one big `LimitComponentTest.java` with 15+ tests; NOT: `LimitsComponentTest` (plural); NOT: `TestLimit` (prefix)

**Test Method:**
- Pattern: `should{Behavior}()`
- Examples: `shouldCreateLimit()`, `shouldRejectInvalidRequest()`
- NOT: `testCreateLimit()`, `createLimit_success()`

**Display Name:**
- Use scenario title from story
- Example: `@DisplayName("Should reject AMOUNT limit without currency")`

---

## Code Templates

### Test Class Template (one small class per sub-feature)

Create **one test class per sub-feature** (e.g. create, update, list, delete), not one large CRUD class. Example for a “create” sub-feature:

```java
package {BASE_PACKAGE}.limit;

import static org.assertj.core.api.Assertions.assertThat;

import {BASE_PACKAGE}.BaseComponentTest;
import {BASE_PACKAGE}.HttpTestHelpers.HttpResponse;
import com.example.limits.api.generated.model.CreateLimitRequest;
// ... other imports in Checkstyle order (java.* before org.*)

@DisplayName("Limit Create Component Tests")
class LimitCreateComponentTest extends BaseComponentTest {

    @Test
    @DisplayName("Should create and retrieve limit")
    void shouldCreateAndRetrieveLimit() {
        CreateLimitRequest limitRequest;
        HttpResponse createResponse;
        String limitId;
        HttpResponse getResponse;
        
        given("I prepare valid COUNT limit request", () -> {
            limitRequest = new CreateLimitRequest(
                    "Daily Transfer Limit",
                    Map.of("entity_type", "account", "operation", "TRANSFER"),
                    LimitType.COUNT,
                    BigDecimal.TEN,
                    null,
                    WindowType.FIXED_DAILY,
                    "HARD",
                    null
            );
        });
        
        when("I POST to limits endpoint", () -> {
            createResponse = httpHelpers.post("/api/v1/admin/limits", limitRequest);
        });
        
        then("limit is created successfully", () -> {
            assertThat(createResponse.statusCode()).isEqualTo(201);
            
            // Extract ID from response
            var responseBody = objectMapper.readTree(createResponse.body());
            limitId = responseBody.get("id").asText();
            assertThat(limitId).isNotNull();
        });
        
        and("I can retrieve the created limit", () -> {
            getResponse = httpHelpers.get("/api/v1/admin/limits/" + limitId);
            assertThat(getResponse.statusCode()).isEqualTo(200);
            
            var limit = objectMapper.readTree(getResponse.body());
            assertThat(limit.get("name").asText()).isEqualTo("Daily Transfer Limit");
            assertThat(limit.get("threshold").asInt()).isEqualTo(10);
        });
    }

    @Test
    @DisplayName("Should reject AMOUNT limit without currency")
    void shouldRejectAmountLimitWithoutCurrency() {
        CreateLimitRequest invalidRequest;
        HttpResponse response;
        
        given("I prepare AMOUNT limit without currency", () -> {
            invalidRequest = new CreateLimitRequest(
                    "Test Limit",
                    null,
                    LimitType.AMOUNT,
                    BigDecimal.valueOf(1000),
                    null, // Currency is missing
                    WindowType.FIXED_DAILY,
                    null,
                    null
            );
        });
        
        when("I create limit via POST", () -> {
            response = httpHelpers.post("/api/v1/admin/limits", invalidRequest);
        });
        
        then("request is rejected with 400", () -> {
            assertThat(response.statusCode()).isEqualTo(400);
        });
        
        and("error message indicates currency required", () -> {
            assertThat(response.body())
                .contains("currency is required for AMOUNT limits");
        });
    }

    // Put update scenarios in LimitUpdateComponentTest, list in LimitListComponentTest, etc.
}
```

---

### WireMock Stub Template (with Grey-box Verification)

```java
package {BASE_PACKAGE}.reservation.stubs;

import {BASE_PACKAGE}.WireMockService;
import static com.github.tomakehurst.wiremock.client.WireMock.*;

/**
 * WireMock stubs for External System API.
 */
public class ExternalSystemStubs extends WireMockService {

    private static final String BASE_PATH = "/external-api";

    public ExternalSystemStubs(String wireMockBaseUrl) {
        super(wireMockBaseUrl);
    }

    /**
     * Stub successful balance check.
     */
    public void stubGetAccountBalanceSuccess(String accountId, double balance, String currency) {
        wireMock.register(
            get(urlEqualTo(BASE_PATH + "/accounts/" + accountId + "/balance"))
                .willReturn(aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                        {
                          "accountId": "%s",
                          "balance": %s,
                          "currency": "%s"
                        }
                        """.formatted(accountId, balance, currency)))
        );
    }

    /**
     * Stub account not found (404).
     */
    public void stubGetAccountBalanceNotFound(String accountId) {
        wireMock.register(
            get(urlEqualTo(BASE_PATH + "/accounts/" + accountId + "/balance"))
                .willReturn(aResponse()
                    .withStatus(404)
                    .withBody("""
                        {
                          "error": "Account not found"
                        }
                        """))
        );
    }

    /**
     * Verify balance check was called (grey-box verification).
     */
    public boolean verifyGetAccountBalanceCalled(String accountId, int expectedCount) {
        var requests = wireMock.find(getRequestedFor(
            urlEqualTo(BASE_PATH + "/accounts/" + accountId + "/balance")));
        return requests.size() >= expectedCount;
    }
}
```

### Test with External Service (Grey-box)

```java
@Test
@DisplayName("Should call external service when checking balance")
void shouldCallExternalServiceWhenCheckingBalance() {
    ExternalSystemStubs externalStubs;
    TransferRequest transferRequest;
    HttpResponse response;
    
    given("external service is available", () -> {
        externalStubs = new ExternalSystemStubs(ContainersSetup.getWireMockBaseUrl());
        externalStubs.clearStubs();
        externalStubs.stubGetAccountBalanceSuccess("acc123", 1000.00, "EUR");
    });
    
    when("I submit transfer operation", () -> {
        transferRequest = new TransferRequest("acc123", "acc456", BigDecimal.valueOf(500), "EUR");
        response = httpHelpers.post("/api/v1/operations/transfers", transferRequest);
    });
    
    then("transfer is processed", () -> {
        assertThat(response.statusCode()).isEqualTo(200);
    });
    
    and("external service was called to check balance", () -> {
        // Grey-box: Verify external call happened
        assertThat(externalStubs.verifyGetAccountBalanceCalled("acc123", 1))
            .isTrue();
    });
}
```

---

## Key Principles

### 1. Small Test Classes (Single Responsibility)
**MANDATORY:** One test class = one coherent area of behaviour (sub-feature). Do **not** put all CRUD or all scenarios for a feature in a single large class.

- **Create** a separate class for create + create-time validations (e.g. `LimitCreateComponentTest`).
- **Update, list, delete, lifecycle, threshold adjustment** each get their own class when there are multiple tests.
- Benefits: easy to find tests (“all create tests”), smaller files, clearer scope.
- If a feature has only 1–2 scenarios total, one class is fine; as soon as you have several distinct behaviours, split.

### 2. Strict BDD Fluent API
**MANDATORY:** Every test MUST use `given/when/then/and` pattern.

**❌ BAD:**
```java
@Test
void testCreateLimit() {
    var request = new CreateLimitRequest(...);
    var response = httpHelpers.post("/limits", request);
    assertThat(response.statusCode()).isEqualTo(201);
}
```

**✅ GOOD:**
```java
@Test
void shouldCreateLimit() {
    given("I prepare valid request", () -> { ... });
    when("I POST to limits endpoint", () -> { ... });
    then("limit is created", () -> { ... });
}
```

### 3. Use DTOs for Requests
**MANDATORY:** Always use request DTOs (Data Transfer Objects) from the `com.example.limits.api` package instead of raw JSON strings. This improves type safety and readability.

**❌ BAD: Raw JSON String**
```java
given("I prepare AMOUNT limit without currency", () -> {
    invalidRequest = """
        {
          "name": "Test Limit",
          "limitType": "AMOUNT",
          "threshold": 1000,
          "windowType": "FIXED_DAILY"
        }
        """;
});
when("I create limit via POST", () -> {
    response = httpHelpers.post("/api/v1/admin/limits", invalidRequest);
});
```

**✅ GOOD: Using a Request DTO**
```java
given("I prepare AMOUNT limit without currency", () -> {
    invalidRequest = new CreateLimitRequest(
            "Test Limit",
            null,
            LimitType.AMOUNT,
            BigDecimal.valueOf(1000),
            null, // Currency is missing
            WindowType.FIXED_DAILY,
            null,
            null
    );
});
when("I create limit via POST", () -> {
    response = httpHelpers.post("/api/v1/admin/limits", invalidRequest);
});
```

### 4. Grey-box Verification
- ✅ Test through API endpoints
- ✅ Verify HTTP responses
- ✅ Verify WireMock calls (external services)
- ❌ NO direct database queries
- ❌ NO testing internal components

### 5. Async Wait Strategy
**Use Awaitility for async operations:**

```java
and("operation completes asynchronously", () -> {
    httpHelpers.waitForCondition(
        () -> {
            var statusResponse = httpHelpers.get("/api/v1/operations/" + operationId);
            var status = objectMapper.readTree(statusResponse.body()).get("status").asText();
            return "COMPLETED".equals(status);
        },
        Duration.ofSeconds(10)
    );
});
```

### 6. Full Response Assertion (Field by Field)
**MANDATORY:** Always verify the full response body, field by field. Do not assert only status code or a single field.

**❌ BAD:**
```java
then("limit is created", () -> {
    assertThat(createResponse.statusCode()).isEqualTo(201);
    limitId = objectMapper.readTree(createResponse.body()).get("id").asText();
});
```

**✅ GOOD:**
```java
then("limit is created with full response verified", () -> {
    assertThat(createResponse.statusCode()).isEqualTo(201);
    var body = objectMapper.readTree(createResponse.body());
    limitId = body.get("id").asText();
    assertThat(limitId).isNotNull().isNotEmpty();
    assertThat(body.get("name").asText()).isEqualTo("Daily Transfer Limit");
    assertThat(body.get("limitType").asText()).isEqualTo("COUNT");
    assertThat(body.get("threshold").asInt()).isEqualTo(10);
    assertThat(body.get("version").asInt()).isGreaterThanOrEqualTo(0);
    assertThat(body.get("createdAt")).isNotNull();
    assertThat(body.get("updatedAt")).isNotNull();
    // ... all relevant fields
});
```

- For **success responses**: assert every field that the API returns (id, name, status, version, timestamps, nested objects, etc.).
- For **paged responses**: assert structure (content, page, size, totalElements, totalPages) and that each item in content has the expected fields. Use `Assertion.assertPagedResponse(body)` from the shared assertions class.

### 7. Single Expected HTTP Status
**MANDATORY:** Assert exactly one expected status code. Do not use `isIn(200, 204)` or similar; the test must expect a single, well-defined outcome.

**❌ BAD:**
```java
assertThat(deleteResponse.statusCode()).isIn(200, 204);
assertThat(getResponse.statusCode()).isIn(200, 404);
```

**✅ GOOD:**
```java
assertThat(deleteResponse.statusCode()).isEqualTo(204);
assertThat(getResponse.statusCode()).isEqualTo(200);
```

If the API contract allows multiple outcomes (e.g. 200 vs 404 after delete), choose one expected behaviour for the test and assert that only (or split into two tests).

### 8. ErrorResponse Schema
When the test expects an error (4xx), the response body must follow the standard ErrorResponse schema. Assert all fields.

**Schema (example):**
```json
{
  "timestamp": "2026-02-05T13:02:55.675516419Z",
  "ref": "7b7df5ef-ea70-4075-b1df-e6e052efa25d",
  "code": "OLA-202",
  "message": "Invalid CEL condition expression: ..."
}
```

**Assertion:** Use the shared assertions class (do not add private helpers in the test class):

```java
// In test — use Assertion
then("request fails with full error response", () -> {
    assertThat(response.statusCode()).isEqualTo(400);
    JsonNode body = objectMapper.readTree(response.body());
    Assertion.assertErrorResponse(body, ErrorCode.INVALID_CEL_EXPRESSION, "expected phrase in message");
});
```

- Error codes are defined in `ErrorCode` (e.g. `ErrorCode.INVALID_CEL_EXPRESSION`, `ErrorCode.INVALID_RULE_STATE_TRANSITION`).
- `Assertion.assertErrorResponse(body, expectedCode)` verifies timestamp, ref, code, and non-blank message. Pass optional message phrases as additional varargs (e.g. `"Invalid CEL condition expression"`); the message must contain at least one of them.

### 9. Resource Cleanup - CRITICAL ⚠️
**MANDATORY:** Every resource created in a test MUST be tracked for cleanup to prevent test pollution.

**The Problem:**
Tests create resources (limits, rules, etc.) that persist in the database. Without cleanup:
- Later tests see unexpected resources from earlier tests
- Tests fail with "Expected 1 limit but got 3 limits"
- Tests become flaky and order-dependent

**The Solution:**
Always use test helpers with cleanup tracking, and ALWAYS call `trackForCleanup()` when creating resources via direct HTTP POST.

#### Pattern 1: Using Test Helpers (Preferred)
```java
@DisplayName("Limit Create Component Tests")
class LimitCreateComponentTest extends BaseComponentTest {

    private LimitTestDataHelper limitHelper;

    @BeforeEach
    void initHelper() {
        this.limitHelper = new LimitTestDataHelper(httpHelpers, objectMapper);
    }

    @AfterEach  // ✅ REQUIRED: Cleanup after each test
    void cleanupTestData() {
        if (limitHelper != null) {
            limitHelper.cleanup();
        }
    }

    @Test
    void shouldCreateLimit() {
        given("I prepare valid request", () -> {
            // Using helper - automatic tracking ✅
            LimitResponse limit = limitHelper.createLimit(request);
        });
    }
}
```

#### Pattern 2: Direct HTTP POST - MUST Track Manually
```java
@Test
void shouldCreateLimitWithSpecialValidation() {
    HttpResponse response;
    
    when("I create limit via direct HTTP POST", () -> {
        response = httpHelpers.post("/api/v1/admin/limits", 
            objectMapper.writeValueAsString(request));
    });
    
    then("limit is created successfully", () -> {
        assertThat(response.statusCode()).isEqualTo(201);
        LimitResponse limit = objectMapper.readValue(response.body(), LimitResponse.class);
        
        // ✅ CRITICAL: Track for cleanup!
        limitHelper.trackForCleanup(limit.getId().toString());
        
        // Continue with assertions...
        assertThat(limit.getName()).isEqualTo("...");
    });
}
```

#### ❌ WRONG: Creating resources without cleanup tracking
```java
@Test
void shouldCreateLimit() {
    response = httpHelpers.post("/api/v1/admin/limits", request);
    assertThat(response.statusCode()).isEqualTo(201);
    
    LimitResponse limit = objectMapper.readValue(response.body(), LimitResponse.class);
    // ❌ BAD: Limit not tracked - will pollute other tests!
    assertThat(limit.getName()).isEqualTo("Test Limit");
}
```

#### Real-World Impact Example
```java
// LimitCreateComponentTest creates limit with broad pattern:
@Test
void shouldCreateRollingLimit() {
    var request = new CreateLimitRequest()
        .name("Rolling 7D Limit")
        .tagPattern(Map.of("entity_type", "account"))  // ⚠️ Matches ALL operations!
        .windowType(WindowType.ROLLING_7_D)
        // ...
    
    response = httpHelpers.post("/api/v1/admin/limits", request);
    LimitResponse limit = objectMapper.readValue(response.body(), LimitResponse.class);
    
    // ❌ BAD: Not tracked - this limit will match operations in ALL later tests!
}

// Later, in ReservationComponentTest:
@Test
void shouldCheckOneLimit() {
    // This test creates its own limit and expects to check only 1 limit
    var reserveResponse = reserve(...);
    
    // ❌ FAILS: Expected 1 limit but got 3 (sees the orphaned limit above + others)
    assertThat(reserveResponse.getLimitsChecked()).hasSize(1);
}
```

#### Cleanup Rules
1. **Always use helpers when available** - They handle tracking automatically
2. **When using direct HTTP POST:**
   - Parse the response to get the created resource ID
   - Call `helper.trackForCleanup(resourceId)` immediately
   - Do this in the `then()` block after asserting 201/200
3. **Always add `@AfterEach cleanup()`** - Without this, tracking doesn't help
4. **Test helpers available:**
   - `LimitTestDataHelper` - for limits
   - `RuleTestDataHelper` - for rules  
   - `CounterTestDataHelper` - for counters (delegates to limit/rule helpers)
   - `ReservationTestDataHelper` - for reservations (delegates to limit/rule helpers)

#### Checklist for Every Test Class
- [ ] Uses test helper (e.g., `LimitTestDataHelper`, `RuleTestDataHelper`)
- [ ] Has `@BeforeEach` to initialize helper
- [ ] Has `@AfterEach` to call `helper.cleanup()`
- [ ] Every direct HTTP POST that creates a resource calls `helper.trackForCleanup(id)`
- [ ] No orphaned resources left after test runs

**Remember:** One forgotten `trackForCleanup()` call can break dozens of unrelated tests!

---

## Anti-Patterns to Avoid

### ❌ BAD: One large test class per feature
```text
limit/LimitComponentTest.java  (15 tests: create, update, delete, list, threshold, validations...)
```
Makes it hard to find “all create tests” or “all list tests”; class becomes large and mixes responsibilities.

### ✅ GOOD: Small classes per sub-feature
```text
limit/LimitCreateComponentTest.java
limit/LimitUpdateComponentTest.java
limit/LimitListComponentTest.java
limit/LimitThresholdAdjustmentComponentTest.java
```
One class = one area of behaviour; easy to navigate and extend.

### ❌ BAD: Raw JSON strings for requests
```java
var request = """
    { "name": "My Limit" }
    """;
httpHelpers.post("/api/v1/admin/limits", request);
```

### ✅ GOOD: Using DTOs for requests
```java
var request = new CreateLimitRequest("My Limit", ...);
httpHelpers.post("/api/v1/admin/limits", request);
```

### ❌ BAD: Direct database queries
```java
@Test
void shouldCreateLimit() {
    httpHelpers.post("/api/v1/admin/limits", request);
    
    // ❌ BAD: Querying database directly
    var result = jdbcTemplate.query("SELECT * FROM limits WHERE id = ?", limitId);
    assertThat(result).hasSize(1);
}
```

### ✅ GOOD: API verification
```java
@Test
void shouldCreateLimit() {
    var createResponse = httpHelpers.post("/api/v1/admin/limits", request);
    var limitId = extractId(createResponse.body());
    
    // ✅ GOOD: Verify through API
    var getResponse = httpHelpers.get("/api/v1/admin/limits/" + limitId);
    assertThat(getResponse.statusCode()).isEqualTo(200);
}
```

### ❌ BAD: Testing internal layers
```java
@Autowired
LimitFacade limitFacade; // ❌ BAD: Component test shouldn't access internals
```

### ✅ GOOD: Testing through API only
```java
// ✅ GOOD: Only HTTP interactions
var response = httpHelpers.post("/api/v1/admin/limits", request);
```

### ❌ BAD: Standard JUnit style
```java
@Test
void testCreateLimit() {
    // Setup
    var request = new CreateLimitRequest(...);
    // Execute
    var response = httpHelpers.post("/limits", request);
    // Verify
    assertThat(response.statusCode()).isEqualTo(201);
}
```

### ✅ GOOD: BDD fluent style
```java
@Test
void shouldCreateLimit() {
    given("valid request", () -> { ... });
    when("POST to endpoint", () -> { ... });
    then("created successfully", () -> { ... });
}
```

### ❌ BAD: Asserting only status or one field
```java
then("created", () -> {
    assertThat(response.statusCode()).isEqualTo(201);
    id = objectMapper.readTree(response.body()).get("id").asText();
});
```

### ✅ GOOD: Full response asserted
```java
then("created with full response", () -> {
    assertThat(response.statusCode()).isEqualTo(201);
    var body = objectMapper.readTree(response.body());
    id = body.get("id").asText();
    assertThat(body.get("name").asText()).isEqualTo("...");
    assertThat(body.get("version")).isNotNull();
    // ... all returned fields
});
```

### ❌ BAD: Multiple allowed status codes
```java
assertThat(response.statusCode()).isIn(200, 204);
```

### ✅ GOOD: Single expected status
```java
assertThat(response.statusCode()).isEqualTo(204);
```

### ❌ BAD: Error body checked only by substring
```java
assertThat(response.body()).contains("error");
```

### ✅ GOOD: ErrorResponse schema asserted
```java
assertThat(response.statusCode()).isEqualTo(400);
JsonNode body = objectMapper.readTree(response.body());
Assertion.assertErrorResponse(body, ErrorCode.INVALID_CEL_EXPRESSION, "expected phrase");
```

### ❌ BAD: Creating resources without cleanup tracking
```java
@Test
void shouldCreateLimitWithRollingWindow() {
    response = httpHelpers.post("/api/v1/admin/limits", request);
    assertThat(response.statusCode()).isEqualTo(201);
    
    LimitResponse limit = objectMapper.readValue(response.body(), LimitResponse.class);
    // ❌ Missing: limitHelper.trackForCleanup(limit.getId().toString());
    assertThat(limit.getWindowType()).isEqualTo(WindowType.ROLLING_7_D);
}
```

### ✅ GOOD: Tracking all created resources
```java
@Test
void shouldCreateLimitWithRollingWindow() {
    response = httpHelpers.post("/api/v1/admin/limits", request);
    assertThat(response.statusCode()).isEqualTo(201);
    
    LimitResponse limit = objectMapper.readValue(response.body(), LimitResponse.class);
    limitHelper.trackForCleanup(limit.getId().toString()); // ✅ Tracked for cleanup
    assertThat(limit.getWindowType()).isEqualTo(WindowType.ROLLING_7_D);
}
```

### ❌ BAD: Missing @AfterEach cleanup
```java
class LimitCreateComponentTest extends BaseComponentTest {
    private LimitTestDataHelper limitHelper;
    
    @BeforeEach
    void init() {
        limitHelper = new LimitTestDataHelper(httpHelpers, objectMapper);
    }
    // ❌ Missing @AfterEach cleanup() - resources will pollute other tests!
}
```

### ✅ GOOD: Proper cleanup lifecycle
```java
class LimitCreateComponentTest extends BaseComponentTest {
    private LimitTestDataHelper limitHelper;
    
    @BeforeEach
    void init() {
        limitHelper = new LimitTestDataHelper(httpHelpers, objectMapper);
    }
    
    @AfterEach  // ✅ Cleanup after each test
    void cleanupTestData() {
        if (limitHelper != null) {
            limitHelper.cleanup();
        }
    }
}
```

---

## Implementation Checklist

**Before marking test implementation complete, verify:**

### BDD Compliance
- [ ] All tests use `given/when/then/and` pattern
- [ ] No standard JUnit-style tests
- [ ] Step descriptions are clear and descriptive
- [ ] BDD step logging visible in output ([GIVEN]/[WHEN]/[THEN]/[AND] via SLF4J)

### Test Coverage (from Story)
- [ ] Every BDD scenario in story has corresponding test method
- [ ] Happy path tested
- [ ] Validation errors tested
- [ ] Edge cases tested

### API Testing
- [ ] All verifications through HTTP API
- [ ] Request bodies use DTOs
- [ ] HTTP status codes asserted (exactly one expected status per assertion, no `isIn(...)`)
- [ ] Full response body asserted field by field (success and paged responses)
- [ ] Error responses asserted via ErrorResponse schema (timestamp, ref, code, message)
- [ ] JSON response bodies parsed and validated
- [ ] NO direct database queries
- [ ] NO internal layer access (Facade, Repository)

### WireMock (if external services)
- [ ] Stub classes created in `{feature}/stubs/` package
- [ ] Stubs are programmatic Java code (not JSON files)
- [ ] Verification methods included (grey-box)
- [ ] Stubs cleared in `@BeforeEach`

### Test Organization
- [ ] Tests organized by feature (package-by-feature)
- [ ] **Small classes:** one class per sub-feature/responsibility (e.g. `LimitCreateComponentTest`, `LimitListComponentTest`), not one large `{Feature}ComponentTest` with many unrelated tests
- [ ] Test class named `{Feature}{SubFeature}ComponentTest.java` (e.g. `RuleLifecycleComponentTest`)
- [ ] Test methods named `should{Behavior}()`
- [ ] `@DisplayName` matches scenario title from story

### Test Quality
- [ ] Tests pass consistently
- [ ] No flaky tests (deterministic)
- [ ] Clear failure messages

### Resource Cleanup (CRITICAL)
- [ ] Test helper initialized in `@BeforeEach` (e.g., `LimitTestDataHelper`)
- [ ] `@AfterEach cleanup()` method calls `helper.cleanup()`
- [ ] Every direct HTTP POST that creates a resource calls `helper.trackForCleanup(id)`
- [ ] Verified no orphaned resources by running test twice (second run should pass)

### Async Operations
- [ ] Awaitility used (not Thread.sleep)
- [ ] Reasonable timeouts (10-30 seconds)
- [ ] Fast polling intervals (100-200ms)

---

## Usage Example

```bash
# After infrastructure is ready and API layer exists:

# 1. Read user story scenarios
cat docs/stories/PALI-191-limits-crud.md

# 2. Create one or more small test classes per sub-feature (e.g. limit/LimitCreateComponentTest.java, limit/LimitListComponentTest.java)

# 3. Map each scenario to a test method in the appropriate class using DTOs

# 4. Run tests
./gradlew componentTest --tests "LimitCreateComponentTest"

# 5. Verify all scenarios pass
```

---

## References

- Component Test Infrastructure Agent: Must be run first (one-time setup)
- User Stories: `docs/stories/` - Source of BDD scenarios
- REST API Developer Agent: Creates Controllers and .http files
