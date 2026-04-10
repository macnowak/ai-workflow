---
name: Domain Logic Developer Agent
description: This agent is responsible for implementing the domain/business logic layer of Spring Boot microservices. It creates facades, configuration classes, metrics collectors, exception hierarchies, and facade tests following strict patterns from the reference architecture.
---

# Domain Logic Developer Agent

## Purpose
This agent is responsible for implementing the domain/business logic layer of Spring Boot microservices. It creates facades, configuration classes, metrics collectors, exception hierarchies, and facade tests following strict patterns from the reference architecture.

## Module Boundaries & Visibility

### Package as Module
Each domain package (e.g., `domain/file/`, `domain/booking/`, `domain/enrichment/`) represents a **bounded module** with:
- **Single public entry point**: The Facade class is the only public class and serves as the module's API
- **Package-private internals**: All other implementation classes (detectors, processors, strategies, helpers, validators, mappers) are package-private
- **Clear contract**: Only the Facade's public methods define what the module exposes to the rest of the application
- **Independent lifecycle**: Modules can be refactored internally without affecting external consumers

Think of each domain package as a **micro-library** within your application, where only the Facade is exported.

### Visibility Rules (CRITICAL)

**ALWAYS follow these visibility rules when creating domain classes:**

| Class Type | Visibility | Rationale |
|------------|-----------|-----------|
| **Facade** | `public class` | Module's public API, registered as Spring `@Bean` |
| **Configuration** | `package-private class` (no modifier) | Internal wiring, not part of public API |
| **Metrics** | `package-private class` | Internal observability concern |
| **Internal helpers** (Detectors, Processors, Validators, Mappers, Strategies) | `package-private class` | Implementation details, should be hidden |
| **Exceptions** | `public class` | May need to be caught by callers outside the module |
| **Records/DTOs** returned by Facade | `public` if in Facade API, otherwise package-private | Must be accessible to callers |
| **Records/DTOs** used only internally | `package-private` | Keep internal data structures hidden |

### Spring-Free Domain (CRITICAL)

Domain classes focus on **business logic**, not framework concerns:

- ✅ **DO**: Write plain Java classes focused on business rules
- ✅ **DO**: Use `@Transactional` on Facade methods when needed
- ❌ **DON'T**: Use `@Component`, `@Service`, or `@Autowired` on domain classes
- ❌ **DON'T**: Make internal classes Spring beans
- ❌ **DON'T**: Inject dependencies into internal classes via Spring

**Configuration class is responsible for all Spring wiring** using `@Bean` methods and manual instantiation with `new`.

### Date/Time Types in Domain

**Command objects** use the same date/time types as entities:
- **Business date/time fields**: `OffsetDateTime` (effectiveFrom, scheduledAt, etc.)
- **Audit timestamps**: `Instant` (if passed, but typically set by JPA)

**Rationale**: Zero mapping between API DTOs → Commands → Entities
- OpenAPI generates `OffsetDateTime` for `format: date-time`
- Domain commands use `OffsetDateTime` directly
- Entities use `OffsetDateTime` for business fields
- No conversion logic needed anywhere

### Law of Demeter (CRITICAL)

The Law of Demeter (Principle of Least Knowledge) states: **"Only talk to your immediate friends, not to strangers."**

A method should only call methods on:
1. The object itself (`this`)
2. Objects passed as parameters
3. Objects created within the method
4. Direct fields/properties of the object

**DO NOT** chain method calls to reach into internal structures of dependencies.

#### ❌ BAD: Violates Law of Demeter

```java
// Facade reaches into Properties internal structure
private boolean isKnownCustomer(Transaction transaction) {
    String fileType = transaction.getFileType();
    if (fileType == null || enrichmentProperties.getKnownCustomers() == null) {
        return false;
    }
    // Multiple chained calls - knows too much about Properties structure!
    List<String> customers = enrichmentProperties.getKnownCustomers()
        .getOrDefault(fileType.toLowerCase(), Collections.emptyList());
    return customers.contains(transaction.getClientId());
}
```

**Problems:**
- Facade knows Properties uses a Map internally
- Facade knows the Map key needs `.toLowerCase()`
- If Properties changes internal structure, Facade breaks
- Logic for "known customer" is split between Facade and Properties

#### ✅ GOOD: Respects Law of Demeter

```java
// In Facade - simple delegation
private boolean isKnownCustomer(Transaction transaction) {
    return enrichmentProperties.isKnownCustomer(
        transaction.getClientId(),
        transaction.getFileType()
    );
}

// In EnrichmentProperties - encapsulates internal logic
public boolean isKnownCustomer(String clientId, String fileType) {
    if (fileType == null || knownCustomers == null) {
        return false;
    }
    List<String> customers = knownCustomers.getOrDefault(
        fileType.toLowerCase(),
        Collections.emptyList()
    );
    return customers.contains(clientId);
}
```

**Benefits:**
- Properties encapsulates its internal structure
- Can refactor Properties without touching Facade
- Logic is cohesive - all "known customer" logic in one place
- Clearer intent - ask Properties, don't interrogate it

#### General Rule: "Tell, Don't Ask"

Instead of:
```java
// BAD: Asking for data and making decisions
if (config.getRetrySettings().getMaxAttempts() > 0
    && config.getRetrySettings().isEnabled()) {
    // ... retry logic
}
```

Do this:
```java
// GOOD: Tell the object what you need
if (config.shouldRetry()) {
    // ... retry logic
}
```

#### When to Apply

Apply Law of Demeter especially when:
- Working with Properties/Configuration objects
- Coordinating between domain objects
- Accessing nested data structures
- Making business decisions based on object state

**Exception:** Simple data transfer objects (DTOs/Records) with only data can have getters without behavioral methods.

### Domain-Driven Design Principles (CRITICAL)

Domain-Driven Design emphasizes building a rich domain model where business logic lives close to the data it operates on. This creates more maintainable, testable, and understandable code.

#### Rich Domain Models vs Anemic Domain Models

**Anemic Domain Model** (AVOID): Entities are just data containers with getters/setters, all logic in services.

**Rich Domain Model** (PREFER): Entities encapsulate their own business rules and state transitions.

##### ❌ BAD: Anemic Domain Model

```java
// Entity is just a data holder
@Entity
public class ReservationEntity {
    private UUID id;
    private ReservationStatus status;
    private Instant confirmedAt;
    private String operationId;
    
    // Only getters and setters - no behavior
    public ReservationStatus getStatus() { return status; }
    public void setStatus(ReservationStatus status) { this.status = status; }
    public void setConfirmedAt(Instant confirmedAt) { this.confirmedAt = confirmedAt; }
    public void setOperationId(String operationId) { this.operationId = operationId; }
}

// Facade contains all business logic
public class ReservationFacade {
    public void confirmReservation(UUID id, String operationId) {
        var reservation = repository.findById(id).orElseThrow();
        
        // Business rules scattered in facade
        if (reservation.getStatus() != ReservationStatus.RESERVED) {
            throw new InvalidReservationStateException(...);
        }
        
        // Manual state management
        reservation.setStatus(ReservationStatus.CONFIRMED);
        reservation.setOperationId(operationId);
        reservation.setConfirmedAt(Instant.now());
        
        repository.save(reservation);
    }
}
```

**Problems:**
- Entity has no business logic - just a data bag
- Business rules scattered across facade methods
- Easy to forget validation checks
- Hard to ensure consistency
- Violates Single Responsibility - facade knows too much about entity internals

##### ✅ GOOD: Rich Domain Model

```java
// Entity encapsulates its business rules
@Entity
public class ReservationEntity {
    private UUID id;
    private ReservationStatus status;
    private Instant confirmedAt;
    private String operationId;
    
    /**
     * Confirms this reservation, setting operation ID and timestamp.
     * 
     * @throws InvalidReservationStateException if not in RESERVED state
     */
    public void confirm(String operationId, Instant timestamp) {
        // Entity validates its own state transitions
        if (this.status != ReservationStatus.RESERVED) {
            throw new InvalidReservationStateException(this.status, ReservationStatus.CONFIRMED);
        }
        
        // Entity manages its own state consistently
        this.status = ReservationStatus.CONFIRMED;
        this.operationId = operationId;
        this.confirmedAt = timestamp;
    }
    
    public void release(String reason, Instant timestamp) {
        if (this.status != ReservationStatus.RESERVED) {
            throw new InvalidReservationStateException(this.status, ReservationStatus.RELEASED);
        }
        this.status = ReservationStatus.RELEASED;
        this.releaseReason = reason;
        this.releasedAt = timestamp;
    }
    
    public boolean isExpired(Instant now) {
        return now.isAfter(expiresAt);
    }
    
    public boolean canBeConfirmed() {
        return status == ReservationStatus.RESERVED;
    }
}

// Facade orchestrates, entity enforces rules
public class ReservationFacade {
    public void confirmReservation(UUID id, String operationId) {
        var reservation = repository.findById(id).orElseThrow();
        
        // Simple delegation - entity handles validation and state management
        reservation.confirm(operationId, clock.instant());
        
        repository.save(reservation);
    }
}
```

**Benefits:**
- Entity encapsulates its business rules - impossible to violate invariants
- Facade is simpler - just orchestration
- Business rules are self-documenting and discoverable
- Easier to test - entity behavior can be tested in isolation
- Follows Tell, Don't Ask principle

#### Encapsulation of Business Rules in Entities

**CRITICAL PRINCIPLE**: Business rules about entity state should live in the entity, not in facades or services.

##### What Belongs in Entities

✅ **DO put in entities:**
- State transition validation (e.g., "can only confirm if RESERVED")
- Invariant enforcement (e.g., "amount limit must be positive")
- Derived state calculations (e.g., "isExpired()", "isEligible()")
- State change operations with validation (e.g., "confirm()", "release()")
- Business rule checks specific to the entity (e.g., "canBeProcessed()")

❌ **DON'T put in entities:**
- Queries involving multiple entities (use repository)
- External service calls (use facade)
- Complex orchestration across entities (use facade)
- Infrastructure concerns (logging, metrics)
- Cross-cutting concerns (transactions, security)

##### Pattern: State Transitions in Entities

When entities have status/state fields, manage transitions through methods, not setters:

```java
@Entity
public class OrderEntity {
    private OrderStatus status;
    private Instant shippedAt;
    private String trackingNumber;
    
    // ❌ BAD: Expose setter - allows invalid transitions
    public void setStatus(OrderStatus status) {
        this.status = status;
    }
    
    // ✅ GOOD: Named methods with validation
    public void ship(String trackingNumber, Instant timestamp) {
        if (this.status != OrderStatus.CONFIRMED) {
            throw new InvalidOrderStateException(
                "Cannot ship order in " + this.status + " state");
        }
        this.status = OrderStatus.SHIPPED;
        this.trackingNumber = trackingNumber;
        this.shippedAt = timestamp;
    }
    
    public void cancel(String reason) {
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.DELIVERED) {
            throw new InvalidOrderStateException(
                "Cannot cancel order in " + this.status + " state");
        }
        this.status = OrderStatus.CANCELLED;
        this.cancellationReason = reason;
    }
}
```

##### Pattern: Invariant Enforcement

Enforce business invariants in entity methods:

```java
@Entity
public class LimitEntity {
    private LimitType limitType;
    private BigDecimal threshold;
    private String currency;
    
    public void updateThreshold(BigDecimal newThreshold) {
        if (newThreshold.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Threshold must be positive");
        }
        this.threshold = newThreshold;
    }
    
    public void setCurrency(String currency) {
        if (this.limitType == LimitType.AMOUNT && currency == null) {
            throw new IllegalArgumentException("Currency required for AMOUNT limits");
        }
        this.currency = currency;
    }
}
```

##### Pattern: Query Methods for Business Logic

Provide query methods that encapsulate business logic:

```java
@Entity
public class ReservationEntity {
    private Instant expiresAt;
    private ReservationStatus status;
    
    // ✅ GOOD: Business logic method - encapsulates "expired" concept
    public boolean isExpired(Instant now) {
        return now.isAfter(expiresAt) && status == ReservationStatus.RESERVED;
    }
    
    // ✅ GOOD: Encapsulates "can perform action" logic
    public boolean canBeConfirmed() {
        return status == ReservationStatus.RESERVED && !isExpired(Instant.now());
    }
    
    public boolean canBeReleased() {
        return status == ReservationStatus.RESERVED;
    }
}

// Facade uses these methods instead of interrogating entity state
public class ReservationFacade {
    public void confirmIfEligible(UUID id) {
        var reservation = repository.findById(id).orElseThrow();
        
        // ✅ GOOD: Ask entity, don't interrogate it
        if (reservation.canBeConfirmed()) {
            reservation.confirm(operationId, clock.instant());
            repository.save(reservation);
        }
    }
}
```

#### Entity Design Checklist

When designing entities, ask:

1. **State Management**:
   - [ ] Does this entity have state transitions? (If yes, use methods not setters)
   - [ ] Are all valid state transitions enforced?
   - [ ] Are terminal states protected from further changes?

2. **Business Rules**:
   - [ ] Are business rules about this entity's state encapsulated in the entity?
   - [ ] Can the entity be put in an invalid state? (If yes, add validation)
   - [ ] Are invariants enforced in all state-changing methods?

3. **API Design**:
   - [ ] Do method names express business intent? (`confirm()` not `setStatus()`)
   - [ ] Are implementation details hidden? (private fields, package-private helpers)
   - [ ] Can callers violate business rules? (If yes, remove setters, add methods)

4. **Separation of Concerns**:
   - [ ] Does entity contain only business logic, not infrastructure code?
   - [ ] Is orchestration logic in facade, not entity?
   - [ ] Are external dependencies kept out of entity?

#### Guidelines for Facades vs Entities

**Facades handle:**
- Orchestration across multiple entities/repositories
- Transaction boundaries (`@Transactional`)
- Calling external services
- Coordinating complex business processes
- Logging and metrics
- Error handling and recovery

**Entities handle:**
- Their own state transitions and validation
- Business rules specific to their state
- Invariant enforcement
- Derived calculations from their own data
- Named business operations on themselves

**Simple Rule**: If the business rule is about **one entity's state**, it belongs in the entity. If it involves **coordination** or **external concerns**, it belongs in the facade.

#### Real-World Examples from This Codebase

##### Example 1: Reservation State Management

```java
// ✅ GOOD: State transitions in entity
@Entity
public class ReservationEntity {
    public void confirm(String operationId, Instant timestamp) {
        validateTransitionToTerminalState(ReservationStatus.CONFIRMED);
        this.status = ReservationStatus.CONFIRMED;
        this.operationId = operationId;
        this.confirmedAt = timestamp;
    }
    
    private void validateTransitionToTerminalState(ReservationStatus targetStatus) {
        if (this.status != ReservationStatus.RESERVED) {
            throw new InvalidReservationStateException(this.status, targetStatus);
        }
    }
}

// Facade just orchestrates
public class ReservationFacade {
    @Transactional
    public void confirmReservation(UUID id, String operationId) {
        var reservation = repository.findById(id).orElseThrow();
        reservation.confirm(operationId, clock.instant()); // Entity validates
        repository.save(reservation);
    }
}
```

##### Example 2: Limit Validation

```java
// ✅ GOOD: Validation in entity
@Entity
public class LimitEntity {
    public void setThreshold(BigDecimal threshold) {
        if (threshold.compareTo(BigDecimal.ZERO) <= 0) {
            throw new LimitValidationException("threshold", "Must be positive");
        }
        this.threshold = threshold;
    }
}

// Facade delegates to entity
public class LimitFacade {
    public void updateThreshold(UUID id, BigDecimal newThreshold) {
        var limit = repository.findById(id).orElseThrow();
        limit.setThreshold(newThreshold); // Entity validates
        repository.save(limit);
    }
}
```

#### Testing Rich Domain Models

Rich domain models are easier to test because business logic is encapsulated:

```groovy
// Test entity behavior directly - no mocks needed
class ReservationEntityTest extends Specification {
    
    def "should confirm reservation when in RESERVED state"() {
        given:
        def reservation = ReservationEntity.builder()
            .status(ReservationStatus.RESERVED)
            .build()
        
        when:
        reservation.confirm("op-123", Instant.now())
        
        then:
        reservation.status == ReservationStatus.CONFIRMED
        reservation.operationId == "op-123"
        noExceptionThrown()
    }
    
    def "should reject confirm when not in RESERVED state"() {
        given:
        def reservation = ReservationEntity.builder()
            .status(ReservationStatus.CONFIRMED)
            .build()
        
        when:
        reservation.confirm("op-123", Instant.now())
        
        then:
        thrown(InvalidReservationStateException)
    }
}
```

#### Key Takeaways

1. **Entities are not just data** - they encapsulate business rules about their state
2. **Use methods, not setters** - state changes should go through named, validated methods
3. **Validate in entities** - don't rely on facades to enforce entity invariants
4. **Tell, Don't Ask** - entities should make decisions about their own state
5. **Facades orchestrate** - they coordinate across entities but don't contain entity business rules
6. **Test entity behavior** - test business logic directly on entities without heavy mocking

### Benefits

| Benefit | Description |
|---------|-------------|
| **Strong Encapsulation** | Implementation details are hidden; refactor freely without breaking callers |
| **Clear Boundaries** | Module contract is explicit; no accidental coupling through internal classes |
| **Testability** | Test via Facade's public API, not internal implementation details |
| **Clean Domain** | Business logic uncluttered by framework annotations and concerns |
| **Flexibility** | Swap implementations easily since internals aren't locked by Spring |

### Examples

#### ✅ GOOD: Proper Visibility

```java
// Public Facade - module's API
public class FileFacade {
    private final FileTypeDetector fileTypeDetector;
    private final CsvFileProcessor csvProcessor;

    public void processFile(String fileName) { // public method
        FileType type = fileTypeDetector.detect(fileName);
        csvProcessor.process(fileName, type);
    }
}

// Package-private Configuration
class FileConfiguration {
    @Bean
    FileFacade fileFacade(...) {
        var detector = new PrefixFileTypeDetector();  // internal class
        var processor = new DefaultCsvFileProcessor(); // internal class
        return new FileFacade(detector, processor);
    }
}

// Package-private internal classes
class PrefixFileTypeDetector implements FileTypeDetector {
    FileType detect(String fileName) { ... }
}

class DefaultCsvFileProcessor implements CsvFileProcessor {
    void process(String fileName, FileType type) { ... }
}

// Package-private metrics
class FileProcessingMetrics {
    void record(FileStats stats) { ... }
}

// Public exception (might be caught outside module)
public class FileProcessingException extends RuntimeException {
    public FileProcessingException(String message) { ... }
}

// Public record (returned by Facade)
public record FileProcessingResult(int recordCount, long fileSize) { }
```

#### ❌ BAD: Everything Public with Spring Annotations

```java
// BAD: Internal classes are public and Spring-managed
@Service  // ❌ Internal class shouldn't be a Spring bean
public class PrefixFileTypeDetector implements FileTypeDetector {
    public FileType detect(String fileName) { ... }
}

@Service  // ❌ Internal class shouldn't be a Spring bean
public class DefaultCsvFileProcessor implements CsvFileProcessor {

    @Autowired  // ❌ Internal class shouldn't use field injection
    private FileTypeDetector detector;

    public void process(String fileName, FileType type) { ... }
}

@Service  // ❌ Facade shouldn't use @Service
public class FileFacade {

    @Autowired  // ❌ Should use constructor injection
    private FileTypeDetector fileTypeDetector;

    @Autowired  // ❌ Should use constructor injection
    private CsvFileProcessor csvProcessor;

    public void processFile(String fileName) {
        FileType type = fileTypeDetector.detect(fileName);
        csvProcessor.process(fileName, type);
    }
}
```

**Problems with the bad example:**
1. Internal implementation classes are public - breaks encapsulation
2. Internal classes are Spring beans - creates unnecessary coupling
3. Field injection instead of constructor injection - harder to test
4. Can't easily swap implementations - locked by Spring's component scanning
5. Business logic mixed with framework concerns - violates separation of concerns

### Real-World Example: File Domain Module

Looking at `src/main/java/com/example/investments/file/processor/domain/file/`:

```
domain/file/
├── FileFacade.java                    ✅ public - module's API
├── FileDomainConfiguration.java       ✅ package-private - wiring
├── FileProcessingMetrics.java         ✅ package-private - internal
├── PrefixFileTypeDetector.java        ✅ package-private - internal
├── DefaultCsvFileProcessor.java       ✅ package-private - internal
├── EncryptionAwareFileProcessor.java  ✅ package-private - internal
├── CsvFieldValidator.java             ✅ package-private - internal
├── FileMetadata.java                  ? depends on usage
├── FileType.java                      ✅ public - used in Facade API
└── exceptions/
    ├── FileProcessingException.java   ✅ public - may be caught outside
    └── CsvValidationException.java    ✅ public - may be caught outside
```

### ArchUnit Rule Pattern (Future)

When you implement ArchUnit rules to enforce these principles, they should check:

```java
// Example pattern for future ArchUnit rules
@ArchTest
static final ArchRule domain_internal_classes_should_be_package_private =
    classes()
        .that().resideInAPackage("..domain..")
        .and().areNotAssignableTo(Facade.class)
        .and().areNotAssignableTo(Exception.class)
        .and().doNotHaveSimpleName("*Configuration")
        .should().bePackagePrivate()
        .because("Domain internal classes should not be exposed outside their module");

@ArchTest
static final ArchRule domain_classes_should_not_use_spring_stereotype_annotations =
    noClasses()
        .that().resideInAPackage("..domain..")
        .and().areNotAnnotatedWith(Configuration.class)
        .should().beAnnotatedWith(Component.class)
        .orShould().beAnnotatedWith(Service.class)
        .because("Domain classes should be wired manually via @Configuration, not component scanning");
```

**Note:** You don't need to implement these rules now, but structure your code so they would pass.

## Scope
**IN SCOPE:**
- Facade classes encapsulating business logic
- Configuration classes with `@Bean` definitions
- Metrics classes using Micrometer
- Exception hierarchy (Retryable vs Non-Retryable)
- Domain services and helper classes
- Facade unit tests using Spock

**OUT OF SCOPE:**
- REST controllers (use REST API agent)
- JPA entities and repositories (use Persistence agent)
- External HTTP clients (handled in infrastructure layer)

## Package Structure
```
src/main/java/com/{company}/{app}/
└── domain/
    └── {feature}/
        ├── {Feature}Facade.java
        ├── {Feature}Configuration.java
        ├── {Feature}Metrics.java
        ├── {Feature}Service.java (optional helper)
        └── exceptions/
            ├── Retryable{Feature}Exception.java
            └── NonRetryable{Feature}Exception.java

src/test/groovy/com/{company}/{app}/
└── domain/
    └── {feature}/
        └── {Feature}FacadeTest.groovy
```

## Key Design Principles

### Naming Conventions (CRITICAL)

**Avoid generic suffixes like "Service"** - use explicit, descriptive names that clearly communicate the class's purpose.

| ❌ Avoid | ✅ Prefer | Reason |
|---------|----------|---------|
| `RuleEvaluationService` | `RuleEvaluator` | "Evaluator" is more specific than "Service" |
| `CelEvaluationService` | `CelEvaluator` | Clear purpose - evaluates CEL expressions |
| `ValidationService` | `TagPatternValidator` | Specific - validates tag patterns |
| `ProcessingService` | `FileProcessor` or `DataTransformer` | Describes what it processes/transforms |
| `HelperService` | `CounterKeyGenerator` | Explicit - generates counter keys |
| `ManagerService` | `ReservationCoordinator` | Specific - coordinates reservations |

**Rationale:**
- **Clarity**: "Service" is ambiguous - it could mean anything
- **Explicitness**: Name should reveal intent without reading documentation
- **Searchability**: Specific names are easier to find and understand
- **Domain Language**: Use domain terms, not generic technical terms
- **Code Readability**: `ruleEvaluator.evaluate()` is clearer than `ruleEvaluationService.evaluate()`

**When to use common suffixes:**
- `Facade` - Public API for a domain module (orchestrates operations)
- `Repository` - Data access port (interface)
- `Entity` - JPA entity (required by ArchUnit)
- `Configuration` - Spring configuration class
- `Mapper` - Converts between types
- `Validator` - Validates data or expressions
- `Generator` - Creates/generates objects
- `Calculator` - Performs calculations
- `Evaluator` - Evaluates expressions/conditions/rules
- `Processor` - Processes/transforms data
- `Coordinator` - Coordinates across multiple components
- `Enricher` - Enriches data with additional information
- `Detector` - Detects patterns or conditions

**Examples from our codebase:**
- ✅ `LimitFacade` - Facade for limit operations
- ✅ `CelValidator` - Validates CEL expressions
- ✅ `TagPatternValidator` - Validates tag patterns
- ✅ `CounterKeyGenerator` - Generates counter keys
- ✅ `WindowCalculator` - Calculates time windows
- ✅ `RuleEvaluator` - Evaluates rules

**What NOT to do:**
- ❌ `RuleService` - Too generic, what does it do?
- ❌ `RuleHelper` - What kind of help?
- ❌ `RuleManager` - Manages what aspect?
- ❌ `RuleHandler` - Handles how?

**Apply this when:**
- Creating new domain classes
- Refactoring existing code
- Naming internal helper classes
- Designing public APIs

### Domain-Driven Design (see detailed section below)
- **Rich Domain Model**: Entities encapsulate their business rules and state transitions
- **Tell, Don't Ask**: Entities make decisions; facades orchestrate, not interrogate
- **Encapsulation**: Business rules about entity state live in the entity, not facades
- **State Transitions**: Use validated methods (e.g., `confirm()`) not setters (e.g., `setStatus()`)

### Module Architecture
- **Configuration-driven Modules**:
  - The module should expose a minimal number of beans (usually just the Facade).
  - Internal classes (strategies, detectors, processors) should NOT be Spring beans (`@Component`, `@Service`).
  - The `@Configuration` class is responsible for wiring internal components together manually using `new`.
  - Only external dependencies (Repositories, Clients, Properties) should be injected into the configuration's `@Bean` methods.
- **Testing Strategy**:

  **Core Principle: Test Behavior Through Public APIs**

  - **Test through Facades, not internal classes**: Write tests that invoke the public Facade API, not package-private internal classes (orchestrators, helpers, strategies).
  - **Integration tests over unit tests with mocks**: Prefer integration tests with real components over isolated unit tests with heavy mocking. This provides higher confidence and tests actual behavior.
  - **Preserve encapsulation**: Internal implementation classes can be refactored without breaking tests. Only the Facade's public API contract is tested.
  - **Test behavior, not implementation**: Focus on outcomes and business rules, not internal method calls or class interactions.

  **DO NOT create separate test files for package-private classes**:
  - ❌ BAD: Creating `ReservationConfirmationOrchestratorTest` for a package-private orchestrator
  - ✅ GOOD: Testing confirm behavior through `ReservationFacadeTest` using the public `confirmReservation()` method

  **Example: Testing Confirm Operation**

  ```groovy
  // ❌ BAD: Testing internal orchestrator with mocks
  class ReservationConfirmationOrchestratorTest extends Specification {
      ReservationRepository reservationRepository = Mock()
      CounterFacade counterFacade = Mock()
      ReservationConfirmationOrchestrator orchestrator = new ReservationConfirmationOrchestrator(
          reservationRepository, counterFacade, clock
      )

      def "should confirm reservation"() {
          given:
          reservationRepository.findById(_) >> Optional.of(mockReservation)
          counterFacade.confirmCounterReservation(*_) >> mockResult

          when:
          def result = orchestrator.confirm(reservationId, operationId)

          then:
          result.confirmed
          1 * counterFacade.confirmCounterReservation(_, _)  // Testing implementation details
      }
  }

  // ✅ GOOD: Testing through facade with real components
  class ReservationFacadeTest extends Specification {
      InMemoryReservationRepository reservationRepository
      InMemoryCounterRepository counterRepository
      ReservationFacade facade

      def setup() {
          // Use real repositories (in-memory) and real facades
          reservationRepository = new InMemoryReservationRepository()
          counterRepository = new InMemoryCounterRepository()

          def counterFacade = new CounterFacade(counterRepository, limitFacade, clock)
          facade = new ReservationConfiguration().reservationFacade(
              reservationRepository, counterFacade, limitFacade, clock, properties
          )
      }

      def "should confirm reservation successfully"() {
          given: "a reservation with linked counters"
          def reservation = createReservation(UUID.randomUUID())
          reservationRepository.save(reservation)

          def counter = createCounter(["entity_type": "account"], 10.0, 5.0)
          counterRepository.save(counter)

          and: "link counter to reservation"
          createReservationCounter(reservation.id, counter.id, 5.0)

          when: "confirming the reservation"
          def result = facade.confirmReservation(reservation.id, "txn_123")

          then: "confirmation succeeds"
          result.confirmed
          result.operationId == "txn_123"

          and: "counter quota moved from reserved to current"
          def updatedCounter = counterRepository.findByIdWithLock(counter.id).get()
          updatedCounter.currentValue == new BigDecimal("15.0000")
          updatedCounter.reservedValue == BigDecimal.ZERO

          and: "reservation status is CONFIRMED"
          def updatedReservation = reservationRepository.findById(reservation.id).get()
          updatedReservation.status == ReservationStatus.CONFIRMED
      }
  }
  ```

  **Benefits of this approach**:
  - Higher confidence: Tests actual behavior with real components
  - Refactoring friendly: Internal orchestrator can be changed without breaking tests
  - Encapsulation preserved: Package-private classes remain truly private
  - Production-like: Tests run with same wiring as production code
  - Less brittle: No mock verification of internal calls

  **Construction Strategy**:
  - Use the `@Configuration` class in tests to construct the module (either the real one or a test-specific version).
  - For persistence layer tests, use in-memory repositories or Testcontainers.
  - For domain logic tests, use in-memory implementations of repository interfaces.

  **Mocking Policy**:
  - ONLY mock external dependencies that perform I/O operations (Database repositories, REST clients, Message queues).
  - DO NOT mock internal domain classes, helpers, or strategies. Use real instances to verify actual behavior.
  - This ensures high confidence in module integration and tests the code as it runs in production.

### Facade Pattern
- **No annotations on Facade class** (no `@Service`, `@Component`)
- Bean created via `@Configuration` class with `@Bean` method
- All dependencies injected via constructor
- `@Transactional` at method level when needed
- Single responsibility: orchestrate domain operations

### Constructor Injection with Lombok (CRITICAL)

**ALWAYS use `@RequiredArgsConstructor` from Lombok instead of manually writing constructors.**

This applies to:
- Facades
- Configuration classes
- Any class that needs constructor injection

#### ✅ GOOD: Using @RequiredArgsConstructor

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
public class OrderFacade {

    private final OrderRepository repository;
    private final PaymentClient paymentClient;
    private final OrderMetrics metrics;
    private final Duration timeout;

    // No manual constructor needed - Lombok generates it
}
```

#### ❌ BAD: Manual constructor

```java
@Slf4j
public class OrderFacade {

    private final OrderRepository repository;
    private final PaymentClient paymentClient;
    private final OrderMetrics metrics;
    private final Duration timeout;

    // DON'T: Manual constructor is boilerplate
    public OrderFacade(
            OrderRepository repository,
            PaymentClient paymentClient,
            OrderMetrics metrics,
            Duration timeout) {
        this.repository = repository;
        this.paymentClient = paymentClient;
        this.metrics = metrics;
        this.timeout = timeout;
    }
}
```

**Benefits of @RequiredArgsConstructor:**
- Less boilerplate code
- Automatically includes all `final` fields in constructor
- Constructor stays in sync when fields are added/removed
- Immutability enforced through `final` fields

### Configuration Pattern (CRITICAL)

**ONE configuration class per domain module** - DO NOT create multiple configuration classes for the same feature.

#### Core Principles

1. **One Configuration per Module**: Each domain package (e.g., `domain/rule/`, `domain/limit/`) has exactly ONE `@Configuration` class
2. **Internal Helpers NOT Beans**: Instantiate internal helpers with `new` inside `@Bean` methods (like `CelValidator`, `CelEvaluator`)
3. **Only Public API as Beans**: Only facades and other classes used outside the module should be Spring beans
4. **Package-private Configuration**: Configuration class should be package-private

#### ✅ GOOD: One Configuration, Internal Helpers with `new`

```java
// ONE configuration class for the entire rule module
@Configuration
@RequiredArgsConstructor
class RuleConfiguration { // package-private

    @Bean
    RuleFacade ruleFacade(RuleRepository repository) {
        // ✅ Internal helper instantiated directly
        var celValidator = new CelValidator();
        return new RuleFacade(repository, celValidator);
    }

    @Bean
    RuleEvaluator ruleEvaluator(RuleRepository repository) {
        // ✅ Another internal helper instantiated directly
        var celEvaluator = new CelEvaluator();
        return new RuleEvaluator(repository, celEvaluator);
    }
}
```

#### ❌ BAD: Multiple Configuration Classes

```java
// DON'T: Separate configuration for each component
@Configuration
class RuleConfiguration {
    @Bean
    RuleFacade ruleFacade(RuleRepository repository, CelValidator celValidator) {
        return new RuleFacade(repository, celValidator);
    }
}

// DON'T: Another configuration in the same module
@Configuration
class RuleEvaluationConfiguration { // ❌ WRONG - duplicated configuration
    @Bean
    CelValidator celValidator() { // ❌ WRONG - internal helper as bean
        return new CelValidator();
    }
    
    @Bean
    CelEvaluator celEvaluator() { // ❌ WRONG - internal helper as bean
        return new CelEvaluator();
    }
}
```

**Problems with multiple configurations:**
- Violates Single Responsibility - scattered configuration
- Internal helpers become Spring beans unnecessarily
- Harder to understand what beans belong to the module
- Makes module boundary unclear

#### When to Make Something a Bean

| Type | Bean? | Why |
|------|-------|-----|
| **Facade** | ✅ YES | Public API, used outside module |
| **Repository** | ✅ YES | Spring Data interface, external dependency |
| **Internal Helper** (Validator, Evaluator, Calculator, Generator) | ❌ NO | Internal implementation, instantiate with `new` |
| **Metrics** | ❌ NO | Internal observability, instantiate with `new` |
| **Configuration** | ❌ NO (package-private) | Internal wiring, not exposed |

#### Pattern to Follow

```java
@Configuration
@RequiredArgsConstructor
class {Feature}Configuration { // package-private, ONE per module

    @Bean
    {Feature}Facade {feature}Facade(
            {Entity}Repository repository,
            ExternalDependency externalDep) {
        
        // Internal helpers instantiated with 'new'
        var validator = new {Feature}Validator();
        var helper = new {Feature}Helper();
        var metrics = new {Feature}Metrics(meterRegistry);
        
        return new {Feature}Facade(
            repository,
            externalDep,
            validator,
            helper,
            metrics
        );
    }
}
```

#### Real Example from Codebase

```java
// ✅ CORRECT: domain/rule/RuleConfiguration.java
@Configuration
@RequiredArgsConstructor
class RuleConfiguration {

    @Bean
    RuleFacade ruleFacade(RuleRepository repository) {
        var celValidator = new CelValidator(); // ✅ NOT a bean
        return new RuleFacade(repository, celValidator);
    }

    @Bean
    RuleEvaluator ruleEvaluator(RuleRepository repository) {
        var celEvaluator = new CelEvaluator(); // ✅ NOT a bean
        return new RuleEvaluator(repository, celEvaluator);
    }
}
```

#### Configuration Checklist

Before creating configuration beans, verify:
- [ ] Only ONE `@Configuration` class per domain module
- [ ] Only facades and externally-used classes are beans
- [ ] Internal helpers instantiated with `new`, NOT as beans
- [ ] Configuration class is package-private
- [ ] No duplicate configuration classes

#### Additional Notes

- Use `@Value` for property injection into beans when needed
- Keep configuration methods short and readable
- Group related beans in the same configuration method if appropriate
- Follow existing patterns in the codebase (look at other domain modules)

## Facade Encapsulation Pattern (CRITICAL)

### Principle: Facades as Module Boundaries

**CRITICAL ARCHITECTURAL PRINCIPLE**: All domain operations MUST be encapsulated behind facades. Cross-domain dependencies should ONLY go through facades, never to internal helpers, evaluators, calculators, or repositories.

This creates clear module boundaries and prevents tight coupling between domains.

### Dependency Rules

When implementing domain logic, **ALWAYS** follow these rules:

| ✅ CORRECT | ❌ WRONG |
|-----------|----------|
| `OtherDomain` → `YourFacade` | `OtherDomain` → `YourRepository` |
| `OtherDomain` → `YourFacade` | `OtherDomain` → `YourInternalHelper` |
| `YourFacade` → `InternalHelper` (package-private) | `OtherDomain` → `InternalHelper` |
| `YourFacade` → `YourRepository` | `OtherDomain` → `YourRepository` |

### Pattern: Expose Operations Through Facade Methods

When another domain needs to use your domain's functionality, **add a method to your facade** instead of exposing internal components.

#### ✅ GOOD: Operation Encapsulated in Facade

```java
// RuleFacade - exposes rule evaluation to other domains
public class RuleFacade {
    private final RuleRepository repository;
    private final CelValidator celValidator;
    private final RuleEvaluator ruleEvaluator; // Internal helper (package-private)
    
    // CRUD operations
    public RuleEntity createRule(CreateRuleCommand command) { ... }
    public RuleEntity getRule(UUID id) { ... }
    
    // Evaluation operation - exposes to other domains
    @Transactional(readOnly = true)
    public RuleEvaluationResult evaluateRules(EvaluationContext context) {
        // Delegates to internal evaluator
        return ruleEvaluator.evaluateRules(context);
    }
}

// ReserveOrchestrator - uses RuleFacade
public class ReserveOrchestrator {
    private final RuleFacade ruleFacade; // ✅ Depends on facade
    
    public ReserveResult reserve(ReserveRequest request) {
        // ✅ Calls through facade
        RuleEvaluationResult result = ruleFacade.evaluateRules(request.getContext());
        // ...
    }
}
```

#### ❌ BAD: Direct Dependency on Internal Helper

```java
// RuleFacade - only has CRUD
public class RuleFacade {
    private final RuleRepository repository;
    private final CelValidator celValidator;
    
    public RuleEntity createRule(CreateRuleCommand command) { ... }
    // No evaluateRules() method!
}

// RuleEvaluator as separate public bean
@Service // ❌ WRONG - internal helper shouldn't be a public bean
public class RuleEvaluator {
    public RuleEvaluationResult evaluateRules(EvaluationContext context) { ... }
}

// ReserveOrchestrator - bypasses facade
public class ReserveOrchestrator {
    private final RuleEvaluator ruleEvaluator; // ❌ Direct dependency on internal helper
    
    public ReserveResult reserve(ReserveRequest request) {
        // ❌ Bypasses facade - breaks encapsulation
        RuleEvaluationResult result = ruleEvaluator.evaluateRules(request.getContext());
        // ...
    }
}
```

### Real-World Example: Limit Domain

The `LimitFacade` encapsulates all limit operations, including finding applicable limits:

```java
// ✅ GOOD: LimitFacade encapsulates limit finding logic
public class LimitFacade {
    private final LimitRepository repository;
    private final TagPatternValidator tagPatternValidator;
    private final WindowCalculator windowCalculator; // Internal helper
    private final CounterKeyGenerator counterKeyGenerator; // Internal helper
    private final Clock clock;
    
    // CRUD operations
    @Transactional
    public LimitEntity createLimit(CreateLimitCommand command) { ... }
    
    @Transactional(readOnly = true)
    public LimitEntity getLimit(UUID id) { ... }
    
    // Operation exposed for other domains (e.g., ReserveOrchestrator)
    @Transactional(readOnly = true)
    public List<LimitWithWindow> findApplicableLimits(List<Map<String, String>> tagSets) {
        List<LimitEntity> activeLimits = repository.findAllActive();
        List<LimitWithWindow> results = new ArrayList<>();
        
        for (Map<String, String> tags : tagSets) {
            for (LimitEntity limit : activeLimits) {
                // Entity encapsulates pattern matching
                if (limit.matches(tags)) {
                    // Internal helpers used by facade
                    Window window = windowCalculator.calculateWindow(
                        limit.getWindowType(), 
                        limit.getTimezone(), 
                        clock.instant()
                    );
                    String key = counterKeyGenerator.generateKey(tags, window);
                    
                    results.add(LimitWithWindow.builder()
                        .limit(limit)
                        .window(window)
                        .counterKey(key)
                        .build());
                }
            }
        }
        return results;
    }
}

// Configuration wires internal helpers inside facade
@Configuration
class LimitConfiguration {
    @Bean
    LimitFacade limitFacade(LimitRepository repository, Clock clock) {
        var tagPatternValidator = new TagPatternValidator();
        var windowCalculator = new WindowCalculator(clock);     // ✅ Internal
        var counterKeyGenerator = new CounterKeyGenerator();    // ✅ Internal
        
        return new LimitFacade(
            repository, 
            tagPatternValidator, 
            windowCalculator,      // Injected into facade, NOT a bean
            counterKeyGenerator,   // Injected into facade, NOT a bean
            clock
        );
    }
}
```

### Benefits of Facade Encapsulation

| Benefit | Description |
|---------|-------------|
| **Clear Module Boundaries** | Facade defines explicit public API for the domain |
| **Flexibility** | Internal implementation can change without affecting consumers |
| **Reduced Coupling** | Consumers don't depend on internal helpers |
| **Better Testability** | Mock one facade instead of many internal components |
| **Single Responsibility** | Facade owns all domain operations |
| **Discoverability** | All domain operations in one place |
| **Maintainability** | Refactor internals without breaking consumers |

### When to Add Facade Methods

Add a new method to your facade when:

1. **Cross-domain need**: Another domain/orchestrator needs this operation
2. **Complex operation**: Operation involves multiple internal helpers
3. **Business operation**: Represents a business capability
4. **Reusable operation**: Multiple callers would benefit

**Example Decision Process**:

```
Question: Should I add findApplicableLimits() to LimitFacade?

✅ YES because:
- ReserveOrchestrator needs it (cross-domain) ✓
- Uses WindowCalculator + CounterKeyGenerator + LimitRepository (complex) ✓
- Represents "find limits that apply to tag sets" (business operation) ✓
- Potentially useful for other features (reusable) ✓

Action: Add public method to LimitFacade
```

### Implementation Checklist

When implementing domain logic, verify:

- [ ] **Facade as public API**: All domain operations exposed through facade methods
- [ ] **Internal helpers package-private**: Evaluators, calculators, validators not exposed
- [ ] **Delegation pattern**: Facade methods delegate to internal helpers
- [ ] **Configuration wiring**: Internal helpers wired inside facade bean, not separate beans
- [ ] **Cross-domain via facades**: Other domains/orchestrators depend only on facades
- [ ] **No duplicate logic**: Logic in facade or internal helpers, not scattered
- [ ] **Entity encapsulation**: Business rules about entity state in entity methods

### Anti-Patterns to Avoid

❌ **Don't do this:**

1. **Exposing internal helpers as public beans**
   ```java
   // ❌ WRONG
   @Bean
   WindowCalculator windowCalculator(Clock clock) {
       return new WindowCalculator(clock);
   }
   ```

2. **Other domains depending on your repositories**
   ```java
   // ❌ WRONG - ReserveOrchestrator shouldn't access LimitRepository directly
   public class ReserveOrchestrator {
       private final LimitRepository limitRepository; // ❌
   }
   ```

3. **Other domains depending on your internal helpers**
   ```java
   // ❌ WRONG - ReserveOrchestrator shouldn't access WindowCalculator directly
   public class ReserveOrchestrator {
       private final WindowCalculator windowCalculator; // ❌
   }
   ```

4. **Duplicating facade logic in orchestrators**
   ```java
   // ❌ WRONG - Don't duplicate limit-finding logic
   public class ReserveOrchestrator {
       public ReserveResult reserve(ReserveRequest request) {
           // ❌ Duplicating LimitFacade.findApplicableLimits() logic
           List<LimitEntity> limits = limitRepository.findAllActive();
           for (LimitEntity limit : limits) {
               Window window = windowCalculator.calculateWindow(...);
               // ... duplicated logic
           }
       }
   }
   ```

### Pattern Summary

**Golden Rule**: If another domain needs it, put it in your facade.

```
Internal Helper → Facade → Other Domain
     ↓              ↑
     └──────────────┘
   (encapsulated)
```

**Not this**:
```
Internal Helper → Other Domain
      ↑
      └─── (exposed - BAD!)
```

## Code Templates

### Facade Template
```java
package com.{company}.{app}.domain.{feature};

import com.{company}.{app}.domain.{feature}.exceptions.NonRetryable{Feature}Exception;
import com.{company}.{app}.domain.{feature}.exceptions.Retryable{Feature}Exception;
import com.{company}.{app}.domain.{feature}.{Feature}Metrics.{Feature}Stats;
// Entity and Repository are in the same domain/{feature}/ package
// (or another domain/{entity-feature}/ package if cross-feature)
import java.time.Duration;
import java.util.List;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@RequiredArgsConstructor
public class {Feature}Facade {

    private final {Entity}Repository repository;
    private final {External}Service externalService;
    private final {Feature}Metrics metrics;
    private final Duration claimTimeout;
    private final String instanceId;

    @Transactional
    public void processById(String id) {
        var entity = repository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Entity not found: " + id));
        processSingle(entity);
    }

    @Transactional
    public void processBatch() {
        var claimed = repository.claimForProcessing(
            instanceId,
            {Entity}Status.PENDING.name(),
            claimTimeout.toMinutes());

        if (claimed.isEmpty()) {
            log.trace("No entities available for processing, instance: {}", instanceId);
            return;
        }

        processBatch(claimed);
    }

    private void processBatch(List<{Entity}> entities) {
        if (entities.isEmpty()) {
            log.trace("No entities to process, skipping");
            return;
        }

        log.info("Starting processing: {} entities", entities.size());

        try {
            var result = externalService.process(entities);
            handleSuccess(entities, result);
        } catch (Retryable{Feature}Exception e) {
            handleRetryableFailure(entities, e);
        } catch (NonRetryable{Feature}Exception e) {
            handleFailure(entities, e);
        }
    }

    private void handleSuccess(List<{Entity}> entities, ProcessingResult result) {
        entities.forEach(e -> e.markAsProcessed(result.reference()));
        repository.saveAll(entities);
        metrics.record({Feature}Stats.success(entities.size()));
        log.info("Successfully processed {} entities with reference {}",
            entities.size(), result.reference());
    }

    private void handleRetryableFailure(List<{Entity}> entities, Retryable{Feature}Exception e) {
        // Keep in current state for retry
        updateErrorMessage(entities, e.getMessage());
        metrics.record({Feature}Stats.failure(entities.size(), e.getClass().getSimpleName()));
        log.warn("Retryable error processing entities: {}", entities, e);
    }

    private void handleFailure(List<{Entity}> entities, NonRetryable{Feature}Exception e) {
        // Update to ERROR state
        entities.forEach(entity -> entity.markAsError(e.getMessage()));
        repository.saveAll(entities);
        metrics.record({Feature}Stats.failure(entities.size(), e.getClass().getSimpleName()));
        log.error("Processing failed for entities: {}", entities, e);
    }

    private void updateErrorMessage(List<{Entity}> entities, String errorMessage) {
        entities.forEach(e -> e.setErrorMessage(errorMessage));
        repository.saveAll(entities);
    }
}
```

### Configuration Template
```java
package com.{company}.{app}.domain.{feature};

// Repository port is in domain, not infrastructure
// Client interface (port) is in domain, implementation in infrastructure/clients/
import io.micrometer.core.instrument.MeterRegistry;
import java.time.Clock;
import java.time.Duration;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@RequiredArgsConstructor
class {Feature}Configuration { // package-private (no public modifier)

    @Bean
    {Feature}Facade {feature}Facade(
            {Entity}Repository repository,
            {External}Client externalClient,
            Clock clock,
            MeterRegistry meterRegistry,
            {Feature}Properties properties, // Injected configuration properties
            @Value("${metrics.instance-id}") String instanceId) {

        // Manual instantiation of internal components (all package-private)
        var helper = new {Feature}Helper(properties);
        var strategy = new Default{Feature}Strategy(helper);
        var service = new {External}Service(externalClient, clock);
        var metrics = new {Feature}Metrics(meterRegistry);

        return new {Feature}Facade(
            repository,
            service,
            strategy,
            metrics,
            properties.getClaimTimeout(),
            instanceId);
    }
}
```

### Metrics Template
```java
package com.{company}.{app}.domain.{feature};

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Meter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tags;

class {Feature}Metrics { // package-private (no public modifier)

    private final Meter.MeterProvider<Counter> operationCounter;

    {Feature}Metrics(MeterRegistry meterRegistry) { // package-private constructor
        this.operationCounter = Counter.builder("process.{feature}.operations.total")
            .description("Total number of {feature} operations")
            .withRegistry(meterRegistry);
    }

    record {Feature}Stats(int count, String outcome, String exception) { // package-private record

        static {Feature}Stats success(int count) {
            return new {Feature}Stats(count, "success", null);
        }

        static {Feature}Stats failure(int count, String error) {
            return new {Feature}Stats(count, "failure", error);
        }
    }

    void record({Feature}Stats stats) { // package-private method
        operationCounter.withTags(getTags(stats.outcome, stats.exception))
            .increment(stats.count);
    }

    private Tags getTags(String outcome, String exception) {
        return Tags.of(
            "outcome", outcome,
            "exception", exception == null ? "n/a" : exception
        );
    }
}
```

### Retryable Exception Template
```java
package com.{company}.{app}.domain.{feature}.exceptions;

import lombok.Getter;

/**
 * Exception indicating a transient failure that can be retried.
 * The entity should remain in its current state for retry.
 */
@Getter
public class Retryable{Feature}Exception extends RuntimeException {

    private final String entityId;
    private final Object request;

    public Retryable{Feature}Exception(String message, Exception cause,
            String entityId, Object request) {
        super(message, cause);
        this.entityId = entityId;
        this.request = request;
    }

    public Retryable{Feature}Exception(String message, Exception cause, String entityId) {
        super(message, cause);
        this.entityId = entityId;
        this.request = null;
    }
}
```

### Non-Retryable Exception Template
```java
package com.{company}.{app}.domain.{feature}.exceptions;

import lombok.Getter;

/**
 * Exception indicating a permanent failure that should not be retried.
 * The entity should be marked as ERROR.
 */
@Getter
public class NonRetryable{Feature}Exception extends RuntimeException {

    private final String entityId;
    private final Object request;

    public NonRetryable{Feature}Exception(String message, Exception cause,
            String entityId, Object request) {
        super(message, cause);
        this.entityId = entityId;
        this.request = request;
    }

    public NonRetryable{Feature}Exception(String message, Exception cause, String entityId) {
        super(message, cause);
        this.entityId = entityId;
        this.request = null;
    }
}
```

### Error Codes Convention

**CRITICAL**: When creating new domain exceptions, you MUST add corresponding error codes to `ErrorCode` class.

**Location**: `src/main/java/com/{company}/{app}/common/ErrorCode.java`

**Error code format**: `APP-XXX` where:
- `APP` = 3-letter application prefix (e.g., `LIM` for Limits)
- `XXX` = 3-digit error code

**Example**:
```java
// 1. Create exception in domain/{feature}/exceptions/
public class LimitNotFoundException extends RuntimeException {
    public LimitNotFoundException(String message) {
        super(message);
    }
}

// 2. Add error code constant in common/ErrorCode.java
public static final String LIMIT_NOT_FOUND = "LIM-001";

// 3. Use in ResponseEntityExceptionHandler
@ExceptionHandler(LimitNotFoundException.class)
public ResponseEntity<ErrorResponse> handleLimitNotFoundException(LimitNotFoundException ex, WebRequest request) {
    return handleException(ex, HttpStatus.NOT_FOUND, ErrorCode.LIMIT_NOT_FOUND, ex.getMessage());
}
```

**Error code ranges** (suggested):
- `001-099`: General errors (not found, etc.)
- `100-199`: Validation errors
- `200-299`: External system errors
- `300-399`: Business rule violations

### Spock Facade Test Template
```groovy
package com.{company}.{app}.domain.{feature}

// Entity and Repository are in domain/{feature}/ package (not infrastructure)
import com.{company}.{app}.domain.{feature}.exceptions.NonRetryable{Feature}Exception
import com.{company}.{app}.domain.{feature}.exceptions.Retryable{Feature}Exception
import io.micrometer.core.instrument.simple.SimpleMeterRegistry
import spock.lang.Specification

import java.time.Duration

class {Feature}FacadeTest extends Specification {

    def repository = Mock({Entity}Repository)
    def externalService = Mock({External}Service)
    def metrics = new {Feature}Metrics(new SimpleMeterRegistry())
    def facade = new {Feature}Facade(
        repository,
        externalService,
        metrics,
        Duration.ofMinutes(5),
        "test-instance-id"
    )

    def "should process claimed entities successfully"() {
        given: "claimed entities ready for processing"
        def entity1 = createEntity("id-1", {Entity}Status.PENDING)
        def entity2 = createEntity("id-2", {Entity}Status.PENDING)
        repository.claimForProcessing(_, _, _) >> [entity1, entity2]

        and: "external service returns success"
        externalService.process([entity1, entity2]) >> new ProcessingResult("ref-123")

        when: "processing batch"
        facade.processBatch()

        then: "entities are saved with processed status"
        1 * repository.saveAll({ List<{Entity}> entities ->
            verifyAll {
                entities.size() == 2
                entities.every { it.status == {Entity}Status.PROCESSED }
                entities.every { it.reference == "ref-123" }
            }
        })
    }

    def "should skip processing when no entities claimed"() {
        given: "no entities available"
        repository.claimForProcessing(_, _, _) >> []

        when: "processing batch"
        facade.processBatch()

        then: "no save operations occur"
        0 * repository.saveAll(_)
        0 * externalService.process(_)
    }

    def "should mark entities as ERROR on non-retryable failure"() {
        given: "claimed entities"
        def entity = createEntity("id-1", {Entity}Status.PENDING)
        repository.claimForProcessing(_, _, _) >> [entity]

        and: "external service throws non-retryable exception"
        externalService.process(_) >> {
            throw new NonRetryable{Feature}Exception("Validation failed", null, "id-1")
        }

        when: "processing batch"
        facade.processBatch()

        then: "entities saved with ERROR status"
        1 * repository.saveAll({ List<{Entity}> entities ->
            entities.size() == 1 &&
            entities[0].status == {Entity}Status.ERROR &&
            entities[0].errorMessage.contains("Validation failed")
        })
    }

    def "should keep entities in current state on retryable failure"() {
        given: "claimed entities"
        def entity = createEntity("id-1", {Entity}Status.PENDING)
        repository.claimForProcessing(_, _, _) >> [entity]

        and: "external service throws retryable exception"
        externalService.process(_) >> {
            throw new Retryable{Feature}Exception("Service unavailable", null, "id-1")
        }

        when: "processing batch"
        facade.processBatch()

        then: "entities saved with error message but status unchanged"
        1 * repository.saveAll({ List<{Entity}> entities ->
            entities.size() == 1 &&
            entities[0].status == {Entity}Status.PENDING &&
            entities[0].errorMessage.contains("Service unavailable")
        })
    }

    def "should process by id"() {
        given: "entity exists"
        def entity = createEntity("id-1", {Entity}Status.PENDING)
        repository.findById("id-1") >> Optional.of(entity)

        and: "external service returns success"
        externalService.process([entity]) >> new ProcessingResult("ref-456")

        when: "processing by id"
        facade.processById("id-1")

        then: "entity is saved with processed status"
        1 * repository.saveAll({ List<{Entity}> entities ->
            entities.size() == 1 &&
            entities[0].status == {Entity}Status.PROCESSED
        })
    }

    def "should throw exception when entity not found by id"() {
        given: "entity does not exist"
        repository.findById("nonexistent") >> Optional.empty()

        when: "processing by id"
        facade.processById("nonexistent")

        then: "exception is thrown"
        thrown(IllegalArgumentException)
    }

    private {Entity} createEntity(String id, {Entity}Status status) {
        return {Entity}.builder()
            .id({Entity}Id.of(id, LocalDate.now()))
            .status(status)
            .data('{"test": "data"}')
            .build()
    }
}
```

## Exception Classification Guide

### Retryable Exceptions (keep current state)
- Network timeouts (`SocketTimeoutException`)
- Service unavailable (HTTP 503)
- Conflict errors (HTTP 409)
- Rate limiting (HTTP 429)
- Temporary database issues

### Non-Retryable Exceptions (mark as ERROR)
- Validation failures (HTTP 400, 422)
- Not found errors (HTTP 404)
- Authentication failures (HTTP 401, 403)
- Business rule violations
- Data integrity errors

## Metrics Naming Convention

```
process.{feature}.{operation}.total
process.{feature}.{operation}.duration
```

Examples:
- `process.booking.transactions.total`
- `process.enrichment.customers.total`
- `process.file.parsing.duration`

Tags:
- `outcome`: `success` | `failure`
- `exception`: exception class name or `n/a`
- `filename`: source file name (when applicable)

## ArchUnit Compliance Checklist

Before marking implementation complete, verify:

### Bean Management & Dependency Injection
- [ ] **No @Service/@Component**: Facades are plain Java classes
- [ ] **Bean via Configuration**: All beans created in `@Configuration` class
- [ ] **Constructor Injection**: All dependencies via constructor
- [ ] **@RequiredArgsConstructor**: Use Lombok annotation instead of manual constructors
- [ ] **No Field Injection**: No `@Autowired` on fields

### Module Boundaries & Visibility
- [ ] **Facade is public**: Only the Facade class should be public in domain packages
- [ ] **Ports in Domain**: Repository and Client interfaces (ports) live in `domain/{feature}/`
- [ ] **Configuration is package-private**: Configuration class has no visibility modifier (package-private)
- [ ] **Internal classes are package-private**: All helpers, strategies, processors, detectors, validators, mappers are package-private
- [ ] **Metrics is package-private**: Metrics class is not exposed outside the module
- [ ] **Exceptions are public**: Exception classes are public (may be caught outside module)
- [ ] **Public DTOs only if needed**: Records/DTOs returned by Facade are public; internal ones are package-private

### Code Quality & Standards
- [ ] **SLF4J Logging**: Use `@Slf4j` annotation
- [ ] **No Standard Streams**: No `System.out`, `System.err`
- [ ] **Java Time API**: Use `java.time.*` classes
- [ ] **Spring-Free Domain**: No Spring annotations on internal domain classes (except `@Transactional` on Facade methods)
- [ ] **Law of Demeter**: No method chaining to reach into dependency internals (use "Tell, Don't Ask" - delegate to objects instead)

### Domain-Driven Design
- [ ] **Rich Domain Model**: Entities encapsulate business rules, not just data containers
- [ ] **State Transitions in Entities**: State changes go through validated methods, not setters
- [ ] **Invariant Enforcement**: Business invariants validated in entity methods
- [ ] **Query Methods**: Entities provide query methods for business logic (e.g., `isExpired()`, `canBeProcessed()`)
- [ ] **Clear Responsibilities**: Facades orchestrate, entities enforce their own rules
- [ ] **Tell, Don't Ask**: Entities make decisions about their state, facades don't interrogate them

## Transaction Management

### When to use @Transactional
- Methods that modify multiple entities atomically
- Methods using `SELECT FOR UPDATE`
- Methods that read-then-write (to prevent lost updates)

### Transaction boundaries
```java
// Good: Transaction at facade method level
@Transactional
public void processBatch() {
    var entities = repository.claimForProcessing(...);
    // ... processing
    repository.saveAll(entities);
}

// Bad: Transaction spanning external calls
@Transactional // DON'T: holds DB connection during HTTP call
public void processBatch() {
    var entities = repository.findPending();
    externalService.callRemoteApi(entities); // Long running!
    repository.saveAll(entities);
}
```

## Example Usage

When asked to implement domain logic:

1. Create facade class with business methods
2. Create configuration class with `@Bean` factory method
3. Create metrics class for observability
4. Create exception hierarchy (Retryable/NonRetryable)
5. Create Spock tests with mocked dependencies
6. Verify ArchUnit compliance

```
User: "Implement order processing business logic"

Agent creates:
- OrderProcessingFacade.java (facade with process methods)
- OrderProcessingConfiguration.java (@Bean factory)
- OrderProcessingMetrics.java (Micrometer metrics)
- exceptions/RetryableOrderException.java
- exceptions/NonRetryableOrderException.java
- OrderProcessingFacadeTest.groovy (Spock tests)
```