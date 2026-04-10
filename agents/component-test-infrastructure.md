---
name: Component Test Infrastructure Agent (v2)
description: Sets up component testing infrastructure for Spring Boot REST APIs. Creates TestContainers configuration, base test classes, HTTP utilities, and Gradle setup. ONE-TIME setup per project.
---

# Component Test Infrastructure Agent (v2)

## Guard: Skip if infrastructure already exists

**FIRST:** Check whether component-test infrastructure is already in place. If it is, **do nothing** — make no file changes, add no tasks. This agent is intended to run at most once in the project lifecycle.

**Consider infrastructure present when ALL of the following exist:**

- `src/componentTest/` source set (directory and Gradle `sourceSets.componentTest` / `componentTest` task in `build.gradle.kts`)
- Base classes: `BddScenario.java`, `BaseComponentTest.java`, `Assertion.java`, `HttpTestHelpers.java`, `ContainersSetup.java`, `WireMockService.java`, `TestObjectMapper.java` under `src/componentTest/java/...`
- Test resources: e.g. `src/componentTest/resources/logback-test.xml` and `junit-platform.properties`

**How to check (examples):**
- `ls src/componentTest/java/*/BaseComponentTest.java src/componentTest/java/*/ContainersSetup.java 2>/dev/null`
- Grep/build: `componentTest` source set and task in `build.gradle.kts`

If all of the above are present, **skip execution** and report briefly that infrastructure is already in place. Do not create or modify any files.

---

## What is a Component Test?

Component tests are **grey-box end-to-end tests** that verify REST APIs in production-like environment:

**✅ WHAT IT DOES:**
- Tests application via HTTP REST API endpoints
- Runs application in Docker container (TestContainers)
- Uses real PostgreSQL in container
- Mocks external services with WireMock
- Verifies HTTP responses (status, JSON body)
- Verifies external service calls (WireMock verification)

**❌ WHAT IT DOESN'T DO:**
- Query database directly (test through API only)
- Test internal layers (Facade, Repository) directly
- Test database infrastructure (connection pools, etc.)

**WHEN TO USE:**
- REST API layer exists (Controllers + .http files)
- User story has BDD scenarios in `docs/stories/`
- Complex workflows or external integrations

---

## Prerequisites Validation

**BEFORE starting, verify these conditions:**

### 1. API Layer exists
```bash
# Check for Controllers
find src/main/java -name "*Controller.java" -path "*/api/controller/*"

# Check for .http files
find http/ -name "*.http"
```
**If empty:** API layer not ready. Run REST API Developer Agent first.

### 2. Application can be containerized
```bash
./gradlew tasks | grep bootJar
```
**If not found:** Gradle bootJar task missing.

### 3. User stories exist
```bash
ls docs/stories/*.md
```
**If empty:** Create user stories with BDD scenarios first.

---

## Scope

**IN SCOPE:**
- Gradle configuration (source sets, dependencies, tasks)
- TestContainers setup (PostgreSQL, WireMock, Application)
- Base classes (BddScenario, BaseComponentTest, HttpTestHelpers)
- WireMock base service
- Test resources (logback, junit properties, reportportal.properties)

**OUT OF SCOPE:**
- Writing actual test scenarios (use Component Test Developer Agent)
- Project-specific WireMock stubs
- Application code changes

---

## Template Variables

Replace these placeholders when implementing:

| Variable | Description | Example |
|----------|-------------|---------|
| `{BASE_PACKAGE}` | Root package | `com.example.limits` |
| `{APP_NAME}` | Application name | `limits-service` |
| `{DB_NAME}` | Database name | `limits_db` |
| `{APP_PORT}` | Application HTTP port | `8080` |
| `{ACTUATOR_PORT}` | Actuator port | `8081` |

---

## Package Structure

```
src/componentTest/
├── java/{BASE_PACKAGE}/
│   ├── BddScenario.java              # Fluent API interface
│   ├── BaseComponentTest.java         # Base class for all tests
│   ├── Assertion.java  # Shared assertions (error response, paged response) — must exist so test classes use it instead of duplicating
│   ├── ContainersSetup.java          # TestContainers configuration
│   ├── HttpTestHelpers.java          # HTTP client utilities (AutoCloseable)
│   ├── TestObjectMapper.java         # Shared ObjectMapper config for tests
│   └── WireMockService.java          # WireMock base class
└── resources/
    ├── logback-test.xml              # Logging configuration
    ├── junit-platform.properties     # JUnit configuration
    └── reportportal.properties       # ReportPortal integration (optional)
```

---

## Code Templates

### 1. BddScenario.java

```java
package {BASE_PACKAGE};

import org.slf4j.LoggerFactory;

/**
 * Fluent API for BDD-style test scenarios.
 * Enforces given/when/then/and pattern.
 * Step descriptions are logged via SLF4J; Logback ReportPortalAppender sends them to RP.
 */
public interface BddScenario {

    default void given(String description, ThrowingRunnable action) {
        logStep("[GIVEN]", description);
        executeStep(description, action);
    }

    default void when(String description, ThrowingRunnable action) {
        logStep("[WHEN]", description);
        executeStep(description, action);
    }

    default void then(String description, ThrowingRunnable action) {
        logStep("[THEN]", description);
        executeStep(description, action);
    }

    default void and(String description, ThrowingRunnable action) {
        logStep("[AND]", description);
        executeStep(description, action);
    }

    default void logStep(String prefix, String description) {
        LoggerFactory.getLogger(getClass()).info("{}: {}", prefix, description);
    }

    default void executeStep(String description, ThrowingRunnable action) {
        try {
            action.run();
        } catch (Throwable e) {
            throw new AssertionError("Step failed: " + description, e);
        }
    }

    @FunctionalInterface
    interface ThrowingRunnable {
        void run() throws Throwable;
    }
}
```

### 2. HttpTestHelpers.java

```java
package {BASE_PACKAGE};

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.hc.client5.http.classic.methods.*;
import org.apache.hc.client5.http.impl.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.core5.http.ContentType;
import org.apache.hc.core5.http.io.entity.EntityUtils;
import org.apache.hc.core5.http.io.entity.StringEntity;
import org.awaitility.Awaitility;

import java.io.IOException;
import java.time.Duration;
import java.util.function.Supplier;

/**
 * HTTP client utilities for component tests.
 * Implements AutoCloseable; call close() (e.g. in @AfterEach) to release the HTTP client.
 */
public class HttpTestHelpers implements AutoCloseable {

    private final String baseUrl;
    private final CloseableHttpClient httpClient;
    private final ObjectMapper objectMapper;

    public HttpTestHelpers(String baseUrl, ObjectMapper objectMapper) {
        this.baseUrl = baseUrl;
        this.httpClient = HttpClients.createDefault();
        this.objectMapper = objectMapper;
    }

    @Override
    public void close() {
        try {
            httpClient.close();
        } catch (IOException e) {
            // log if needed
        }
    }

    public HttpResponse get(String path) {
        try {
            var request = new HttpGet(baseUrl + path);
            var response = httpClient.execute(request);
            return new HttpResponse(
                response.getCode(),
                EntityUtils.toString(response.getEntity())
            );
        } catch (IOException e) {
            throw new RuntimeException("GET request failed: " + path, e);
        }
    }

    public HttpResponse post(String path, String jsonBody) {
        try {
            var request = new HttpPost(baseUrl + path);
            request.setEntity(new StringEntity(jsonBody, ContentType.APPLICATION_JSON));
            var response = httpClient.execute(request);
            return new HttpResponse(
                response.getCode(),
                EntityUtils.toString(response.getEntity())
            );
        } catch (IOException e) {
            throw new RuntimeException("POST request failed: " + path, e);
        }
    }

    public HttpResponse put(String path, String jsonBody) {
        try {
            var request = new HttpPut(baseUrl + path);
            request.setEntity(new StringEntity(jsonBody, ContentType.APPLICATION_JSON));
            var response = httpClient.execute(request);
            return new HttpResponse(
                response.getCode(),
                EntityUtils.toString(response.getEntity())
            );
        } catch (IOException e) {
            throw new RuntimeException("PUT request failed: " + path, e);
        }
    }

    public HttpResponse delete(String path) {
        try {
            var request = new HttpDelete(baseUrl + path);
            var response = httpClient.execute(request);
            return new HttpResponse(
                response.getCode(),
                EntityUtils.toString(response.getEntity())
            );
        } catch (IOException e) {
            throw new RuntimeException("DELETE request failed: " + path, e);
        }
    }

    public <T> T parseResponse(String json, Class<T> clazz) {
        try {
            return objectMapper.readValue(json, clazz);
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse response", e);
        }
    }

    /**
     * Wait for condition using Awaitility.
     */
    public void waitForCondition(Supplier<Boolean> condition, Duration timeout) {
        Awaitility.await()
            .atMost(timeout)
            .pollInterval(Duration.ofMillis(100))
            .until(condition::get);
    }

    public static class HttpResponse {
        private final int statusCode;
        private final String body;

        public HttpResponse(int statusCode, String body) {
            this.statusCode = statusCode;
            this.body = body;
        }

        public int statusCode() {
            return statusCode;
        }

        public String body() {
            return body;
        }
    }
}
```

### 3. ContainersSetup.java

```java
package {BASE_PACKAGE};

import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.Network;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.utility.DockerImageName;

/**
 * TestContainers setup for component tests.
 * Singleton containers shared across all test classes.
 */
public class ContainersSetup {

    private static final Network SHARED_NETWORK = Network.newNetwork();

    // PostgreSQL Container
    public static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>(
            DockerImageName.parse("postgres:16-alpine"))
        .withNetwork(SHARED_NETWORK)
        .withNetworkAliases("postgres")
        .withDatabaseName("{DB_NAME}")
        .withUsername("test_user")
        .withPassword("test_password")
        .withReuse(true);

    // WireMock Container
    public static final GenericContainer<?> WIREMOCK = new GenericContainer<>(
            DockerImageName.parse("wiremock/wiremock:3.10.0"))
        .withNetwork(SHARED_NETWORK)
        .withNetworkAliases("wiremock")
        .withExposedPorts(8080)
        .waitingFor(Wait.forHttp("/__admin/health").forPort(8080))
        .withReuse(true);

    // Application Container
    public static final GenericContainer<?> APPLICATION = new GenericContainer<>(
            DockerImageName.parse("{APP_NAME}:latest"))
        .withNetwork(SHARED_NETWORK)
        .withNetworkAliases("app")
        .withExposedPorts({APP_PORT}, {ACTUATOR_PORT})
        .withEnv("DATABASE_URL", "jdbc:postgresql://postgres:5432/{DB_NAME}")
        .withEnv("DATABASE_USERNAME", "test_user")
        .withEnv("DATABASE_PASSWORD", "test_password")
        .withEnv("WIREMOCK_BASE_URL", "http://wiremock:8080")
        .withEnv("S2S_AUTH_SERVER_ENABLED", "false")
        .withEnv("S2S_AUTH_CLIENT_ENABLED", "false")
        .dependsOn(POSTGRES, WIREMOCK)
        .waitingFor(Wait.forHttp("/actuator/health").forPort({ACTUATOR_PORT}))
        .withReuse(true);

    static {
        // Start containers once for all tests
        POSTGRES.start();
        WIREMOCK.start();
        APPLICATION.start();
    }

    public static String getApplicationBaseUrl() {
        return "http://localhost:" + APPLICATION.getMappedPort({APP_PORT});
    }

    public static String getWireMockBaseUrl() {
        return "http://localhost:" + WIREMOCK.getMappedPort(8080);
    }
}
```

### 4. BaseComponentTest.java

```java
package {BASE_PACKAGE};

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;

/**
 * Base class for all component tests.
 * Provides HTTP client, JSON mapper, and BDD step methods (given/when/then/and).
 * Containers are started once via ContainersSetup. Report Portal extension registered for results.
 */
public abstract class BaseComponentTest implements BddScenario {

    protected HttpTestHelpers httpHelpers;
    protected ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        this.objectMapper = TestObjectMapper.create();
        this.httpHelpers = new HttpTestHelpers(ContainersSetup.getApplicationBaseUrl(), objectMapper);
    }

    @AfterEach
    void tearDown() throws Exception {
        if (httpHelpers != null) {
            httpHelpers.close();
        }
    }

    /** Returns a unique name for test data to avoid DB unique constraint conflicts between tests. */
    protected static String uniqueName(String baseName) {
        return baseName + " " + System.nanoTime();
    }
}
```

### 5. TestObjectMapper.java

```java
package {BASE_PACKAGE};

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

/**
 * Shared ObjectMapper configuration for component tests.
 * Single place for JSON (de)serialization settings (e.g. Java 8 date/time).
 */
public final class TestObjectMapper {

    private TestObjectMapper() {
    }

    public static ObjectMapper create() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}
```

### 6. WireMockService.java

```java
package {BASE_PACKAGE};

import com.github.tomakehurst.wiremock.client.WireMock;

/**
 * Base class for WireMock stub implementations.
 * Extend this to create service-specific stub classes.
 */
public abstract class WireMockService {

    protected final WireMock wireMock;

    public WireMockService(String wireMockBaseUrl) {
        this.wireMock = new WireMock(wireMockBaseUrl);
    }

    public void clearStubs() {
        wireMock.resetMappings();
    }
}
```

### 7. Assertion.java

**Required:** A dedicated class for shared assertions so that test classes do not duplicate error/paged response checks. Create this class and use it from all component tests.

```java
package {BASE_PACKAGE};

import static org.assertj.core.api.Assertion.assertThat;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Arrays;

/**
 * Shared assertions for component tests.
 * Use these instead of duplicating error/paged response checks in individual test classes.
 */
public final class Assertion {

    private Assertion() {}

    /**
     * Verifies full error response: code, non-empty timestamp and ref, and optionally that message contains the given phrase (or at least one of the phrases if multiple passed).
     */
    public static void assertErrorResponse(JsonNode body, String expectedCode, String... expectedMessagePhrases) {
        assertThat(body.get("timestamp").asText()).isNotBlank();
        assertThat(body.get("ref").asText()).isNotBlank();
        assertThat(body.get("code").asText()).isEqualTo(expectedCode);
        String message = body.get("message").asText();
        assertThat(message).isNotBlank();
        if (expectedMessagePhrases.length > 0) {
            assertThat(Arrays.stream(expectedMessagePhrases).anyMatch(message::contains))
                .as("Message should contain one of: %s", Arrays.toString(expectedMessagePhrases))
                .isTrue();
        }
    }

    /**
     * Verifies standard paged response structure (content, page, size, totalElements, totalPages).
     */
    public static void assertPagedResponse(JsonNode body) {
        assertThat(body.has("content")).isTrue();
        assertThat(body.has("page")).isTrue();
        assertThat(body.has("size")).isTrue();
        assertThat(body.has("totalElements")).isTrue();
        assertThat(body.has("totalPages")).isTrue();
    }
}
```

---

## Gradle Configuration

### build.gradle.kts additions

```kotlin
// Source set for component tests
sourceSets {
    create("componentTest") {
        compileClasspath += sourceSets.main.get().output
        runtimeClasspath += sourceSets.main.get().output
    }
}

val componentTestImplementation by configurations.getting {
    extendsFrom(configurations.implementation.get())
}

val componentTestRuntimeOnly by configurations.getting {
    extendsFrom(configurations.runtimeOnly.get())
}

dependencies {
    // Component test dependencies
    "componentTestImplementation"("org.junit.jupiter:junit-jupiter:5.10.0")
    "componentTestImplementation"("org.testcontainers:testcontainers:1.21.3")
    "componentTestImplementation"("org.testcontainers:junit-jupiter:1.21.3")
    "componentTestImplementation"("org.testcontainers:postgresql:1.21.3")
    "componentTestImplementation"("org.wiremock:wiremock-standalone:3.13.1")
    "componentTestImplementation"("org.assertj:assertj-core:3.27.3")
    "componentTestImplementation"("org.awaitility:awaitility:4.2.0")
    "componentTestImplementation"("org.apache.httpcomponents.client5:httpclient5:5.4.2")
    "componentTestImplementation"("com.fasterxml.jackson.core:jackson-databind:2.15.2")
    "componentTestImplementation"("com.epam.reportportal:agent-java-junit5:5.4.0")
}

tasks.register<Test>("componentTest") {
    useJUnitPlatform()
    description = "Runs component tests with TestContainers"
    group = "verification"

    testClassesDirs = sourceSets["componentTest"].output.classesDirs
    classpath = sourceSets["componentTest"].runtimeClasspath

    // Ensure application is built before tests
    dependsOn(tasks.bootJar)

    // TestContainers configuration
    systemProperty("testcontainers.reuse.enable", "true")

    // Sequential execution (no parallelism)
    systemProperty("junit.jupiter.execution.parallel.enabled", "false")

    failFast = true

    testLogging {
        showStandardStreams = true
        events = setOf(
            org.gradle.api.tasks.testing.logging.TestLogEvent.PASSED,
            org.gradle.api.tasks.testing.logging.TestLogEvent.FAILED,
            org.gradle.api.tasks.testing.logging.TestLogEvent.SKIPPED
        )
        exceptionFormat = org.gradle.api.tasks.testing.logging.TestExceptionFormat.FULL
    }
}
```

---

## Test Resources

### logback-test.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Reduce TestContainers noise -->
    <logger name="org.testcontainers" level="INFO"/>
    <logger name="com.github.dockerjava" level="WARN"/>
    <logger name="org.apache.http" level="WARN"/>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

### junit-platform.properties

```properties
# Sequential execution (no parallelism)
junit.jupiter.execution.parallel.enabled=false
junit.jupiter.execution.parallel.mode.default=same_thread
```

### reportportal.properties

Utwórz plik `src/componentTest/resources/reportportal.properties` z konfiguracją ReportPortal (opcjonalnie, do raportowania wyników testów do RP):

```properties
rp.attributes=
rp.project=
rp.launch=
rp.endpoint=https\://rp.example.com
rp.api.key=
```

Wartości `rp.project`, `rp.launch` i `rp.api.key` uzupełniane są per środowisko (np. zmienne środowiskowe lub lokalnie – nie commituj klucza API).

---

## Implementation Checklist

**Before marking infrastructure complete, verify:**

### Container Setup
- [ ] PostgreSQL container starts and is reachable
- [ ] WireMock container starts and is reachable
- [ ] Application container starts and passes health check
- [ ] All containers use shared network
- [ ] Container reuse is enabled (faster test runs)

### Base Classes
- [ ] `BddScenario.java` provides given/when/then/and methods; step logging uses text prefixes [GIVEN]/[WHEN]/[THEN]/[AND] (SLF4J)
- [ ] `BaseComponentTest.java` sets up HTTP client and ObjectMapper in @BeforeEach; closes httpHelpers in @AfterEach; provides uniqueName(String)
- [ ] `TestObjectMapper.java` provides shared ObjectMapper config (create())
- [ ] `Assertion.java` exists with assertErrorResponse and assertPagedResponse (shared assertions class)
- [ ] `HttpTestHelpers.java` implements AutoCloseable; constructor takes (baseUrl, ObjectMapper); provides GET/POST/PUT/DELETE and Awaitility wrapper
- [ ] `WireMockService.java` provides base stub functionality

### Gradle Configuration
- [ ] `componentTest` source set created
- [ ] All required dependencies added (`com.epam.reportportal:agent-java-junit5` included)
- [ ] `componentTest` task configured
- [ ] Task depends on `bootJar`
- [ ] Sequential execution enabled (no parallel tests)
- [ ] TestContainers reuse enabled

### Test Resources
- [ ] `logback-test.xml` reduces TestContainers noise
- [ ] `junit-platform.properties` disables parallelism
- [ ] `reportportal.properties` w `src/componentTest/resources/` (structure: rp.attributes, rp.project, rp.launch, rp.endpoint, rp.api.key)

### Validation
- [ ] Run `./gradlew componentTest` (should fail with "no tests found" - expected)
- [ ] Containers start successfully
- [ ] Application health check passes
- [ ] WireMock admin API accessible

---

## Usage Example

After infrastructure is set up:

```bash
# Infrastructure is ready
# Next: Use Component Test Developer Agent to create actual tests
```

---

## References

- Component Test Developer Agent: Use this AFTER infrastructure is complete
- TestContainers Documentation: https://testcontainers.com/
- Awaitility Documentation: https://github.com/awaitility/awaitility
- WireMock Documentation: https://wiremock.org/
