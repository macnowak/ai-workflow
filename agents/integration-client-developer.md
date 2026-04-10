---
name: Integration Client Developer Agent
description: This agent is responsible for implementing REST clients that integrate with external systems. It creates HTTP clients with resilience patterns, DTOs, configuration, and comprehensive tests using MockRestServiceServer following strict patterns from the reference architecture.
---

# Integration Client Developer Agent

## Purpose
This agent is responsible for implementing REST clients that integrate with external systems (internal microservices or third-party APIs). It creates HTTP clients with proper resilience patterns (circuit breaker, retry, timeout), request/response DTOs, configuration beans, and comprehensive tests.

## Scope
**IN SCOPE:**
- REST client classes using RestTemplate or WebClient
- Resilience4j integration (CircuitBreaker, Retry, RateLimiter, Bulkhead)
- Client configuration beans and properties
- Request/Response DTOs
- Exception handling and mapping
- Caching with Spring Cache
- Unit tests using MockRestServiceServer
- Health indicators for external dependencies

**OUT OF SCOPE:**
- REST controllers (use REST API agent)
- Business logic and facades (use Domain Logic agent)
- Database persistence (use Persistence agent)
- Message consumers/producers (different agent)

## Package Structure

Following **Pragmatic Hexagonal Architecture** from ARCHITECTURE.md:

```
src/main/java/com/{company}/{app}/
├── domain/
│   └── {feature}/                              # Domain feature that needs external client
│       └── {ExternalSystem}Client.java         # Port (interface) - defined in domain!
└── infrastructure/
    ├── clients/                                # All client implementations here
    │   └── {external-system}/
    │       ├── Default{ExternalSystem}Client.java  # Implementation of port
    │       ├── {ExternalSystem}ClientConfiguration.java
    │       ├── {ExternalSystem}ClientProperties.java
    │       ├── dto/
    │       │   ├── {Operation}Request.java
    │       │   └── {Operation}Response.java
    │       └── exception/
    │           └── {ExternalSystem}ClientException.java
    └── config/
        └── HttpClientConfiguration.java (shared)

src/test/groovy/com/{company}/{app}/
└── infrastructure/
    └── clients/
        └── {external-system}/
            └── Default{ExternalSystem}ClientTest.groovy
```

**Key principle:** Client interface (port) lives in `domain/{feature}/`, implementation lives in `infrastructure/clients/{system}/`.

## Code Templates

### Client Port Interface Template (in domain)
```java
package com.{company}.{app}.domain.{feature};

import java.util.Optional;

/**
 * Port (interface) for {ExternalSystem} integration.
 *
 * Defined in domain layer - implementation lives in infrastructure/clients/.
 * This interface abstracts the external system from domain logic.
 */
public interface {ExternalSystem}Client {

    /**
     * Retrieves resource by ID.
     */
    Optional<{Response}Response> getById(String id);

    /**
     * Creates a new resource.
     */
    {Response}Response create({Request}Request request);

    /**
     * Updates a resource.
     */
    void update(String id, {Request}Request request);
}
```

### Client Implementation Template (in infrastructure)
```java
package com.{company}.{app}.infrastructure.clients.{system};

import com.{company}.{app}.domain.{feature}.{ExternalSystem}Client;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.retry.Retry;
import java.util.Optional;
import java.util.function.Supplier;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

/**
 * Implementation of {ExternalSystem}Client port.
 *
 * Implements resilience patterns:
 * - Circuit Breaker: Opens after consecutive failures to prevent cascade
 * - Retry: Retries transient failures with exponential backoff
 * - Caching: Reduces load for frequently accessed data
 */
@Slf4j
public class Default{ExternalSystem}Client implements {ExternalSystem}Client {

    private final RestTemplate restTemplate;
    private final {ExternalSystem}ClientProperties properties;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;

    public Default{ExternalSystem}Client(RestTemplate restTemplate,
                                  {ExternalSystem}ClientProperties properties,
                                  CircuitBreaker {system}CircuitBreaker,
                                  Retry {system}Retry) {
        this.restTemplate = restTemplate;
        this.properties = properties;
        this.circuitBreaker = {system}CircuitBreaker;
        this.retry = {system}Retry;
    }

    /**
     * Retrieves resource by ID.
     *
     * IMPORTANT: URL uses placeholder {id} with uriVariables to avoid
     * high-cardinality metrics. Never concatenate IDs directly into URL strings.
     *
     * @param id the resource identifier
     * @return Optional containing the response, or empty if not found
     */
    @Cacheable(value = "{system}Cache", key = "#id", unless = "#result == null")
    public Optional<{Response}Response> getById(String id) {
        log.debug("Calling {external-system} to get resource: {}", id);

        // CORRECT: Use URI template with placeholder
        String url = properties.getBaseUrl() + "/api/v1/resources/{id}";

        Supplier<Optional<{Response}Response>> supplier = () -> {
            try {
                var response = restTemplate.getForObject(url, {Response}Response.class, id);

                if (response != null) {
                    log.debug("Successfully retrieved resource: {}", id);
                    return Optional.of(response);
                }
                log.warn("Empty response for resource: {}", id);
                return Optional.empty();

            } catch (HttpClientErrorException.NotFound e) {
                // 404 is valid business response - resource doesn't exist
                log.info("Resource not found: {}", id);
                return Optional.empty();

            } catch (HttpClientErrorException e) {
                // 4xx errors (except 404) - client error, don't retry
                log.error("Client error from {external-system} for {}: {} - {}",
                    id, e.getStatusCode(), e.getResponseBodyAsString());
                throw new {ExternalSystem}ClientException(
                    "Client error: " + e.getStatusCode(), e, false);

            } catch (HttpServerErrorException e) {
                // 5xx errors - server error, can retry
                log.error("Server error from {external-system} for {}: {}",
                    id, e.getStatusCode());
                throw new {ExternalSystem}ClientException(
                    "Server error: " + e.getStatusCode(), e, true);

            } catch (RestClientException e) {
                // Network errors, timeouts - can retry
                log.error("Connection error to {external-system} for {}: {}",
                    id, e.getMessage());
                throw new {ExternalSystem}ClientException(
                    "Connection error", e, true);
            }
        };

        return executeWithResilience(supplier, id);
    }

    /**
     * Creates a new resource.
     *
     * POST requests typically should NOT be retried to avoid duplicates,
     * unless the operation is idempotent.
     */
    public {Response}Response create({Request}Request request) {
        log.debug("Creating resource in {external-system}: {}", request);

        String url = properties.getBaseUrl() + "/api/v1/resources";

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<{Request}Request> entity = new HttpEntity<>(request, headers);

        Supplier<{Response}Response> supplier = () -> {
            try {
                var response = restTemplate.postForObject(url, entity, {Response}Response.class);
                log.info("Successfully created resource");
                return response;

            } catch (HttpClientErrorException.Conflict e) {
                // 409 Conflict - resource already exists
                log.warn("Resource already exists: {}", e.getResponseBodyAsString());
                throw new {ExternalSystem}ClientException(
                    "Resource already exists", e, false);

            } catch (HttpClientErrorException e) {
                log.error("Client error creating resource: {} - {}",
                    e.getStatusCode(), e.getResponseBodyAsString());
                throw new {ExternalSystem}ClientException(
                    "Failed to create resource", e, false);

            } catch (RestClientException e) {
                log.error("Error creating resource: {}", e.getMessage());
                throw new {ExternalSystem}ClientException(
                    "Connection error", e, true);
            }
        };

        // For non-idempotent operations, consider not using retry
        return circuitBreaker.decorateSupplier(supplier).get();
    }

    /**
     * Updates a resource (idempotent PUT).
     *
     * PUT is idempotent, so retry is safe.
     */
    public void update(String id, {Request}Request request) {
        log.debug("Updating resource {} in {external-system}", id);

        // CORRECT: Multiple placeholders in URL template
        String url = properties.getBaseUrl() + "/api/v1/resources/{id}";

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<{Request}Request> entity = new HttpEntity<>(request, headers);

        Supplier<Void> supplier = () -> {
            try {
                restTemplate.put(url, entity, id);
                log.info("Successfully updated resource: {}", id);
                return null;

            } catch (HttpClientErrorException.NotFound e) {
                log.error("Resource not found for update: {}", id);
                throw new {ExternalSystem}ClientException(
                    "Resource not found: " + id, e, false);

            } catch (RestClientException e) {
                log.error("Error updating resource {}: {}", id, e.getMessage());
                throw new {ExternalSystem}ClientException(
                    "Update failed", e, true);
            }
        };

        executeWithResilience(supplier, id);
    }

    /**
     * Applies resilience patterns (circuit breaker + retry) to the operation.
     */
    private <T> T executeWithResilience(Supplier<T> supplier, String context) {
        Supplier<T> decoratedSupplier = circuitBreaker
            .decorateSupplier(retry.decorateSupplier(supplier));

        try {
            return decoratedSupplier.get();
        } catch ({ExternalSystem}ClientException e) {
            // Re-throw domain exceptions
            throw e;
        } catch (Exception e) {
            log.error("All attempts failed for {}: {}", context, e.getMessage());
            throw new {ExternalSystem}ClientException(
                "Operation failed after retries", e, false);
        }
    }
}
```

### URL Construction Best Practices

**CRITICAL: Avoid High-Cardinality Metrics**

When constructing URLs with dynamic parts (IDs, parameters), ALWAYS use URI templates with placeholders:

```java
// CORRECT - Uses placeholder, RestTemplate handles URI encoding
String url = baseUrl + "/api/v1/customers/{customerId}/orders/{orderId}";
restTemplate.getForObject(url, Response.class, customerId, orderId);

// CORRECT - Using UriComponentsBuilder for complex queries
String url = UriComponentsBuilder
    .fromHttpUrl(baseUrl)
    .path("/api/v1/search")
    .queryParam("status", status)
    .queryParam("page", page)
    .build()
    .toUriString();

// WRONG - String concatenation creates unique URLs for each ID
// This causes metrics explosion in Micrometer/Prometheus
String url = baseUrl + "/api/v1/customers/" + customerId;  // DON'T DO THIS

// WRONG - Template literals with embedded values
String url = baseUrl + "/api/v1/customers/" + customerId + "/orders/" + orderId;
```

Why this matters:
- RestTemplate records metrics per URL pattern
- With placeholders: one metric for `/api/v1/customers/{id}`
- Without: millions of metrics `/api/v1/customers/123`, `/api/v1/customers/456`, etc.
- High cardinality destroys monitoring system performance

### Client Properties Template
```java
package com.{company}.{app}.infrastructure.clients.{system};

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import java.time.Duration;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@Data
@Validated
@ConfigurationProperties(prefix = "{system}-client")
public class {ExternalSystem}ClientProperties {

    /**
     * Base URL of the external system API.
     */
    @NotBlank
    private String baseUrl;

    /**
     * Connection timeout for establishing HTTP connection.
     */
    private Duration connectTimeout = Duration.ofSeconds(5);

    /**
     * Read timeout for waiting for response data.
     */
    private Duration readTimeout = Duration.ofSeconds(30);

    /**
     * Maximum number of connections in pool.
     */
    @Positive
    private int maxConnections = 20;

    /**
     * Maximum connections per route (per host).
     */
    @Positive
    private int maxConnectionsPerRoute = 10;

    /**
     * Circuit breaker configuration.
     */
    private CircuitBreakerProperties circuitBreaker = new CircuitBreakerProperties();

    /**
     * Retry configuration.
     */
    private RetryProperties retry = new RetryProperties();

    /**
     * Cache TTL for cached responses.
     */
    private Duration cacheTtl = Duration.ofMinutes(30);

    @Data
    public static class CircuitBreakerProperties {
        /**
         * Failure rate threshold percentage to open circuit.
         */
        private float failureRateThreshold = 50;

        /**
         * Minimum number of calls before calculating failure rate.
         */
        private int minimumNumberOfCalls = 10;

        /**
         * Sliding window size for failure rate calculation.
         */
        private int slidingWindowSize = 20;

        /**
         * Time to wait in open state before transitioning to half-open.
         */
        private Duration waitDurationInOpenState = Duration.ofSeconds(30);

        /**
         * Number of permitted calls in half-open state.
         */
        private int permittedNumberOfCallsInHalfOpenState = 5;
    }

    @Data
    public static class RetryProperties {
        /**
         * Maximum number of retry attempts (including initial call).
         */
        private int maxAttempts = 3;

        /**
         * Initial wait duration between retries.
         */
        private Duration waitDuration = Duration.ofMillis(500);

        /**
         * Multiplier for exponential backoff.
         */
        private double multiplier = 2.0;
    }
}
```

### Client Configuration Template
```java
package com.{company}.{app}.infrastructure.clients.{system};

import com.{company}.{app}.domain.{feature}.{ExternalSystem}Client;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;
import java.time.Duration;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManager;
import org.apache.hc.core5.util.Timeout;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.ResourceAccessException;
import org.springframework.web.client.RestTemplate;

@Configuration
@EnableConfigurationProperties({ExternalSystem}ClientProperties.class)
class {ExternalSystem}ClientConfiguration {

    @Bean
    {ExternalSystem}Client {system}Client(  // Returns port interface from domain
            {ExternalSystem}ClientProperties properties,
            CircuitBreaker {system}CircuitBreaker,
            Retry {system}Retry) {

        RestTemplate restTemplate = createRestTemplate(properties);

        return new Default{ExternalSystem}Client(  // Instantiates implementation
            restTemplate,
            properties,
            {system}CircuitBreaker,
            {system}Retry
        );
    }

    @Bean
    CircuitBreaker {system}CircuitBreaker(
            {ExternalSystem}ClientProperties properties,
            CircuitBreakerRegistry circuitBreakerRegistry) {

        var cbProps = properties.getCircuitBreaker();

        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(cbProps.getFailureRateThreshold())
            .minimumNumberOfCalls(cbProps.getMinimumNumberOfCalls())
            .slidingWindowSize(cbProps.getSlidingWindowSize())
            .waitDurationInOpenState(cbProps.getWaitDurationInOpenState())
            .permittedNumberOfCallsInHalfOpenState(
                cbProps.getPermittedNumberOfCallsInHalfOpenState())
            .recordExceptions(
                HttpServerErrorException.class,
                ResourceAccessException.class)
            .ignoreExceptions(
                {ExternalSystem}ClientException.class)
            .build();

        return circuitBreakerRegistry.circuitBreaker("{system}", config);
    }

    @Bean
    Retry {system}Retry(
            {ExternalSystem}ClientProperties properties,
            RetryRegistry retryRegistry) {

        var retryProps = properties.getRetry();

        RetryConfig config = RetryConfig.custom()
            .maxAttempts(retryProps.getMaxAttempts())
            .waitDuration(retryProps.getWaitDuration())
            .intervalFunction(attempt ->
                (long) (retryProps.getWaitDuration().toMillis() *
                    Math.pow(retryProps.getMultiplier(), attempt - 1)))
            .retryExceptions(
                HttpServerErrorException.class,
                ResourceAccessException.class)
            .ignoreExceptions(
                {ExternalSystem}ClientException.class)
            .build();

        return retryRegistry.retry("{system}", config);
    }

    private RestTemplate createRestTemplate({ExternalSystem}ClientProperties properties) {
        // Connection pool configuration
        PoolingHttpClientConnectionManager connectionManager =
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(properties.getMaxConnections());
        connectionManager.setDefaultMaxPerRoute(properties.getMaxConnectionsPerRoute());

        // Request configuration with timeouts
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(
                Timeout.of(properties.getConnectTimeout()))
            .setResponseTimeout(
                Timeout.of(properties.getReadTimeout()))
            .build();

        // Build HTTP client
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setDefaultRequestConfig(requestConfig)
            .build();

        // Create RestTemplate with HTTP client
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);

        return new RestTemplate(factory);
    }
}
```

### Client Exception Template
```java
package com.{company}.{app}.infrastructure.clients.{system}.exception;

import lombok.Getter;

/**
 * Exception thrown by {ExternalSystem}Client operations.
 *
 * The retryable flag indicates whether the operation can be safely retried:
 * - true: transient errors (network, 5xx) - retry may succeed
 * - false: permanent errors (4xx, business errors) - retry won't help
 */
@Getter
public class {ExternalSystem}ClientException extends RuntimeException {

    private final boolean retryable;

    public {ExternalSystem}ClientException(String message, Throwable cause, boolean retryable) {
        super(message, cause);
        this.retryable = retryable;
    }

    public {ExternalSystem}ClientException(String message, boolean retryable) {
        super(message);
        this.retryable = retryable;
    }
}
```

### Response DTO Template
```java
package com.{company}.{app}.infrastructure.clients.{system}.dto;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

/**
 * Response DTO for {operation} operation.
 *
 * Uses @JsonIgnoreProperties to handle forward compatibility -
 * new fields added by the external system won't break deserialization.
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public record {Operation}Response(
    String id,
    String status,
    // Add fields matching external API response
    String data
) {}
```

### Request DTO Template
```java
package com.{company}.{app}.infrastructure.clients.{system}.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

/**
 * Request DTO for {operation} operation.
 */
public record {Operation}Request(
    @NotBlank
    String name,

    @NotNull
    String data
) {}
```

### Spock Client Test Template
```groovy
package com.{company}.{app}.infrastructure.clients.{system}

import com.{company}.{app}.domain.{feature}.{ExternalSystem}Client
import com.{company}.{app}.infrastructure.clients.{system}.dto.{Operation}Response
import com.{company}.{app}.infrastructure.clients.{system}.exception.{ExternalSystem}ClientException
import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
import io.github.resilience4j.retry.Retry
import io.github.resilience4j.retry.RetryConfig
import org.springframework.http.HttpMethod
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.test.web.client.MockRestServiceServer
import org.springframework.web.client.RestTemplate
import spock.lang.Specification
import spock.lang.Subject

import java.time.Duration

import static org.springframework.test.web.client.match.MockRestRequestMatchers.*
import static org.springframework.test.web.client.response.MockRestResponseCreators.*

/**
 * Unit test for Default{ExternalSystem}Client using MockRestServiceServer.
 *
 * Test class names MUST end with 'Test' suffix (not 'Spec').
 */
class Default{ExternalSystem}ClientTest extends Specification {

    static final String BASE_URL = "http://test-{system}"

    @Subject
    {ExternalSystem}Client client  // Reference via port interface

    RestTemplate restTemplate
    MockRestServiceServer mockServer
    CircuitBreaker circuitBreaker
    Retry retry
    {ExternalSystem}ClientProperties properties

    def setup() {
        // Create properties
        properties = new {ExternalSystem}ClientProperties()
        properties.baseUrl = BASE_URL

        // Create permissive circuit breaker for testing
        circuitBreaker = CircuitBreaker.of("test", CircuitBreakerConfig.custom()
            .failureRateThreshold(80)
            .slidingWindowSize(10)
            .minimumNumberOfCalls(5)
            .build())

        // Create retry with minimal delays for testing
        retry = Retry.of("test", RetryConfig.custom()
            .maxAttempts(2)
            .waitDuration(Duration.ofMillis(10))
            .build())

        // Create RestTemplate and MockServer
        restTemplate = new RestTemplate()
        mockServer = MockRestServiceServer.createServer(restTemplate)

        // Create client implementation
        client = new Default{ExternalSystem}Client(
            restTemplate,
            properties,
            circuitBreaker,
            retry
        )
    }

    // ===========================================
    // SUCCESS SCENARIOS
    // ===========================================

    def "should successfully retrieve resource by ID"() {
        given: "a valid resource ID"
        def resourceId = "resource-123"

        and: "external system returns valid response"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andExpect(method(HttpMethod.GET))
            .andRespond(withSuccess("""
                {
                    "id": "$resourceId",
                    "status": "ACTIVE",
                    "data": "test-data"
                }
                """, MediaType.APPLICATION_JSON))

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should return the resource"
        result.isPresent()
        result.get().id() == resourceId
        result.get().status() == "ACTIVE"

        and: "request was made correctly"
        mockServer.verify()
    }

    def "should successfully create resource"() {
        given: "a valid request"
        def request = new {Request}Request("test-name", "test-data")

        and: "external system accepts the request"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources"))
            .andExpect(method(HttpMethod.POST))
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(content().json('{"name":"test-name","data":"test-data"}'))
            .andRespond(withSuccess("""
                {
                    "id": "new-123",
                    "status": "CREATED",
                    "data": "test-data"
                }
                """, MediaType.APPLICATION_JSON))

        when: "calling create"
        def result = client.create(request)

        then: "should return created resource"
        result.id() == "new-123"
        result.status() == "CREATED"

        and: "request was made correctly"
        mockServer.verify()
    }

    // ===========================================
    // NOT FOUND (404) SCENARIOS
    // ===========================================

    def "should return empty optional when resource not found (404)"() {
        given: "a non-existent resource ID"
        def resourceId = "non-existent"

        and: "external system returns 404"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withStatus(HttpStatus.NOT_FOUND))

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should return empty optional"
        result.isEmpty()

        and: "request was made"
        mockServer.verify()
    }

    // ===========================================
    // CLIENT ERROR (4xx) SCENARIOS
    // ===========================================

    def "should throw non-retryable exception on 400 Bad Request"() {
        given: "a resource ID"
        def resourceId = "bad-request"

        and: "external system returns 400"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withBadRequest()
                .body('{"error": "Invalid request"}'))

        when: "calling getById"
        client.getById(resourceId)

        then: "should throw non-retryable exception"
        def ex = thrown({ExternalSystem}ClientException)
        !ex.retryable

        and: "no retry attempted (only one request)"
        mockServer.verify()
    }

    def "should throw non-retryable exception on 409 Conflict"() {
        given: "a request that causes conflict"
        def request = new {Request}Request("duplicate", "data")

        and: "external system returns 409"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources"))
            .andRespond(withStatus(HttpStatus.CONFLICT)
                .body('{"error": "Resource already exists"}'))

        when: "calling create"
        client.create(request)

        then: "should throw non-retryable exception"
        def ex = thrown({ExternalSystem}ClientException)
        !ex.retryable
    }

    // ===========================================
    // SERVER ERROR (5xx) AND RETRY SCENARIOS
    // ===========================================

    def "should retry on 500 and succeed on second attempt"() {
        given: "a resource ID"
        def resourceId = "retry-success"

        and: "first request fails, second succeeds"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withServerError())
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess("""
                {"id": "$resourceId", "status": "ACTIVE", "data": "data"}
                """, MediaType.APPLICATION_JSON))

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should return result after retry"
        result.isPresent()
        result.get().id() == resourceId

        and: "both requests were made"
        mockServer.verify()
    }

    def "should exhaust retries and fail on persistent 5xx errors"() {
        given: "a resource ID"
        def resourceId = "always-fails"

        and: "all requests fail with 500"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withServerError())
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withServerError())

        when: "calling getById"
        client.getById(resourceId)

        then: "should throw exception after retries"
        thrown({ExternalSystem}ClientException)

        and: "all retry attempts were made"
        mockServer.verify()
    }

    def "should retry on connection timeout"() {
        given: "a resource ID"
        def resourceId = "timeout-recovery"

        and: "first request times out, second succeeds"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withGatewayTimeout())
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess("""
                {"id": "$resourceId", "status": "ACTIVE", "data": "data"}
                """, MediaType.APPLICATION_JSON))

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should succeed after retry"
        result.isPresent()

        and: "both requests were made"
        mockServer.verify()
    }

    // ===========================================
    // RESPONSE HANDLING SCENARIOS
    // ===========================================

    def "should return empty when response body is null"() {
        given: "a resource ID"
        def resourceId = "empty-response"

        and: "external system returns empty body"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess())

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should return empty optional"
        result.isEmpty()
    }

    def "should handle malformed JSON gracefully"() {
        given: "a resource ID"
        def resourceId = "malformed"

        and: "external system returns invalid JSON (will be retried)"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess("not-valid-json", MediaType.APPLICATION_JSON))
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess("still-not-valid", MediaType.APPLICATION_JSON))

        when: "calling getById"
        client.getById(resourceId)

        then: "should throw exception after retries"
        thrown(Exception)

        and: "retry attempts were made"
        mockServer.verify()
    }

    def "should ignore unknown fields in response (forward compatibility)"() {
        given: "a resource ID"
        def resourceId = "future-fields"

        and: "response contains unknown fields"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess("""
                {
                    "id": "$resourceId",
                    "status": "ACTIVE",
                    "data": "data",
                    "newFieldFromFuture": "should be ignored",
                    "anotherNewField": 12345
                }
                """, MediaType.APPLICATION_JSON))

        when: "calling getById"
        def result = client.getById(resourceId)

        then: "should successfully parse response"
        result.isPresent()
        result.get().id() == resourceId
    }

    // ===========================================
    // REQUEST VERIFICATION SCENARIOS
    // ===========================================

    def "should send correct headers in POST request"() {
        given: "a request"
        def request = new {Request}Request("name", "data")

        and: "mock server expects specific headers"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources"))
            .andExpect(method(HttpMethod.POST))
            .andExpect(header("Content-Type", "application/json"))
            .andRespond(withSuccess('{"id":"1","status":"CREATED","data":"d"}',
                MediaType.APPLICATION_JSON))

        when: "calling create"
        client.create(request)

        then: "headers were sent correctly"
        mockServer.verify()
    }

    def "should properly encode path parameters"() {
        given: "a resource ID with special characters"
        def resourceId = "id-with-special"

        and: "mock server"
        mockServer.expect(requestTo("$BASE_URL/api/v1/resources/$resourceId"))
            .andRespond(withSuccess('{"id":"1","status":"OK","data":"d"}',
                MediaType.APPLICATION_JSON))

        when: "calling getById"
        client.getById(resourceId)

        then: "URL was encoded correctly"
        mockServer.verify()
    }
}
```

### Health Indicator Template
```java
package com.{company}.{app}.infrastructure.clients.{system};

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

/**
 * Health indicator for {ExternalSystem} integration.
 *
 * Reports health based on circuit breaker state:
 * - CLOSED: healthy, requests flowing normally
 * - OPEN: unhealthy, requests being rejected
 * - HALF_OPEN: degraded, testing recovery
 */
@Component
@RequiredArgsConstructor
public class {ExternalSystem}HealthIndicator implements HealthIndicator {

    private final CircuitBreaker {system}CircuitBreaker;

    @Override
    public Health health() {
        CircuitBreaker.State state = {system}CircuitBreaker.getState();
        CircuitBreaker.Metrics metrics = {system}CircuitBreaker.getMetrics();

        Health.Builder builder = switch (state) {
            case CLOSED -> Health.up();
            case OPEN -> Health.down();
            case HALF_OPEN -> Health.status("DEGRADED");
            case DISABLED, FORCED_OPEN -> Health.unknown();
            default -> Health.unknown();
        };

        return builder
            .withDetail("state", state)
            .withDetail("failureRate", metrics.getFailureRate())
            .withDetail("slowCallRate", metrics.getSlowCallRate())
            .withDetail("bufferedCalls", metrics.getNumberOfBufferedCalls())
            .withDetail("failedCalls", metrics.getNumberOfFailedCalls())
            .build();
    }
}
```

## Resilience Patterns Reference

### Circuit Breaker States
```
         ┌──────────────────────────────────────┐
         │            CLOSED                    │
         │   (Normal operation, requests flow)  │
         │                                      │
         │  failure rate > threshold            │
         └────────────────┬─────────────────────┘
                          │
                          ▼
         ┌──────────────────────────────────────┐
         │             OPEN                     │
         │  (Requests rejected immediately)     │
         │                                      │
         │  wait duration elapsed               │
         └────────────────┬─────────────────────┘
                          │
                          ▼
         ┌──────────────────────────────────────┐
         │          HALF_OPEN                   │
         │  (Test requests, check recovery)     │
         │                                      │
         │  success → CLOSED / failure → OPEN   │
         └──────────────────────────────────────┘
```

### Retry with Exponential Backoff
```
Attempt 1: immediate
Attempt 2: wait 500ms
Attempt 3: wait 1000ms (500ms * 2)
Attempt 4: wait 2000ms (500ms * 4)
```

### When to Retry vs Not Retry

| Scenario | Retry? | Reason |
|----------|--------|--------|
| 5xx Server Error | Yes | Transient, may recover |
| Network Timeout | Yes | Transient, may recover |
| Connection Refused | Yes | Service may be restarting |
| 404 Not Found | No | Resource doesn't exist |
| 400 Bad Request | No | Request is invalid |
| 401/403 Auth Error | No | Credentials are wrong |
| 409 Conflict | No | Business constraint violated |
| POST (non-idempotent) | Caution | May create duplicates |
| PUT/DELETE (idempotent) | Yes | Safe to retry |

## Testing Best Practices

### MockRestServiceServer Capabilities
- Verify exact request URL including query parameters
- Verify HTTP method (GET, POST, PUT, DELETE)
- Verify request headers (Content-Type, Authorization)
- Verify request body (exact JSON, JSON path, contains)
- Simulate various response scenarios (success, errors, delays)
- Verify request order and count

### Test Categories to Cover
1. **Happy path**: successful requests with valid responses
2. **Not found**: 404 responses returning empty/null
3. **Client errors**: 400, 401, 403, 409 - non-retryable
4. **Server errors**: 500, 502, 503 - retryable with verification
5. **Timeouts**: connection and read timeouts
6. **Malformed responses**: invalid JSON, missing fields
7. **Forward compatibility**: unknown fields in response
8. **Request verification**: headers, body content, encoding

## Configuration Properties Example

```yaml
# application.yml
{system}-client:
  base-url: https://api.{system}.example.com
  connect-timeout: 5s
  read-timeout: 30s
  max-connections: 20
  max-connections-per-route: 10
  cache-ttl: 30m

  circuit-breaker:
    failure-rate-threshold: 50
    minimum-number-of-calls: 10
    sliding-window-size: 20
    wait-duration-in-open-state: 30s
    permitted-number-of-calls-in-half-open-state: 5

  retry:
    max-attempts: 3
    wait-duration: 500ms
    multiplier: 2.0
```

## ArchUnit Compliance Checklist

Before marking implementation complete, verify:

- [ ] **Port in Domain**: Client interface (port) lives in `domain/{feature}/`
- [ ] **Implementation in Infrastructure**: Client implementation lives in `infrastructure/clients/{system}/`
- [ ] **No Field Injection**: Use constructor injection only
- [ ] **URL Placeholders**: All dynamic URL parts use `{placeholder}` with uriVariables
- [ ] **Test Class Naming**: Test classes MUST use `Test` suffix
- [ ] **SLF4J Logging**: Use `@Slf4j` annotation
- [ ] **No Deprecated APIs**: No deprecated Spring or HTTP client APIs
- [ ] **Java Time API**: Use `java.time.Duration` for timeouts
- [ ] **DTOs use @JsonIgnoreProperties**: For forward compatibility

## Example Usage

When asked to create integration client for an external system:

1. Create client interface (port) in `domain/{feature}/`
2. Create client implementation in `infrastructure/clients/{system}/`
3. Create client properties class
4. Create client configuration with beans
5. Create request/response DTOs
6. Create client exception class
7. Create health indicator
8. Create comprehensive test class

```
User: "Create REST client to call Payment Gateway API"

Agent creates:
In domain/reservation/ (or the feature that needs the client):
- PaymentGatewayClient.java (port interface)

In infrastructure/clients/payment-gateway/:
- DefaultPaymentGatewayClient.java (implementation)
- PaymentGatewayClientProperties.java
- PaymentGatewayClientConfiguration.java
- dto/CreatePaymentRequest.java
- dto/PaymentResponse.java
- exception/PaymentGatewayClientException.java
- PaymentGatewayHealthIndicator.java

In test/infrastructure/clients/payment-gateway/:
- DefaultPaymentGatewayClientTest.groovy
```
