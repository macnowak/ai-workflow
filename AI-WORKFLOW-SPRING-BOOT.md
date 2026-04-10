# AI-Assisted Development Workflow for Spring Boot Microservices

This document describes the AI-assisted development workflow for Spring Boot microservices using Claude Code and specialized implementation agents.

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPECIFICATION PHASE                           │
│  /spec {PROJECT} "{Feature Name}"                                │
│  → docs/specifications/SPEC-{PROJECT}-{NUM}-{name}.md           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORY DECOMPOSITION                           │
│  /story {PROJECT}-{NUM} "{Story Description}"                    │
│  → docs/stories/{PROJECT}-{NUM}-{short-desc}.md                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION PLANNING                       │
│  /plan {PROJECT}-{NUM}                                           │
│  → docs/plans/PLAN-{PROJECT}-{NUM}-{feature}.md                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION PHASES                         │
│  /implement Phase 1  → Persistence (@persistence-developer)     │
│  /implement Phase 2  → Integration Clients (@integration-client)│
│  /implement Phase 3  → Domain Logic (@domain-developer)         │
│  /implement Phase 4  → REST API (@rest-api-developer)           │
│  /implement Phase 5  → Integration & Testing                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CODE REVIEW & QUALITY GATE                    │
│  @code-reviewer → Automated review of all changes               │
│  → Verify architecture patterns, coding standards, tests        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VERIFICATION & DELIVERY                       │
│  ./gradlew check → Run tests + ArchUnit validation              │
│  git add . && git commit -m "feat(PROJECT-NUM): description"    │
│  git push → Push to remote                                      │
└─────────────────────────────────────────────────────────────────┘
```

## The Workflow Phases

### Specification Phase
Create a high-level PRD-style specification document that describes:
- Product overview and business context
- Features and functional requirements
- Data models and entity relationships
- API contracts and endpoints
- Technical architecture decisions

### Story Decomposition Phase
Break down the specification into implementable user stories with:
- Feature statement (As a... I want... So that...)
- BDD acceptance criteria (Given/When/Then)
- Business rules and edge cases
- Links to parent specification

### Implementation Phase
Convert each story into a phased implementation plan and execute:
- **Phase 1**: Persistence Layer (entities, repositories, Liquibase)
- **Phase 2**: Integration Clients (REST clients for external systems)
- **Phase 3**: Domain Logic (facades, services, metrics)
- **Phase 4**: REST API (controllers, DTOs, .http files)
- **Phase 5**: Integration & Testing (manual verification)

## Command Reference

| Command | Purpose | Output |
|---------|---------|--------|
| `/spec {PROJECT} "{Name}"` | Create PRD-style specification | `docs/specifications/SPEC-{PROJECT}-{NUM}-{name}.md` |
| `/story {PROJECT}-{NUM} "{Description}"` | Create user story with BDD criteria | `docs/stories/{PROJECT}-{NUM}-{short-desc}.md` |
| `/plan {PROJECT}-{NUM}` | Generate implementation plan | `docs/plans/PLAN-{PROJECT}-{NUM}-{feature}.md` |
| `/implement Phase N` | Execute implementation phase | Code, tests, docs per agent |
| `/review` or `/review quick` | Automated code review (thorough or quick mode) | Review report with issues and recommendations |

## Implementation Phases Detailed

### Phase 1: Persistence Layer

**Agent**: `.claude/agents/persistence-developer.md`

**Creates**:
- JPA entities with `@Entity`, `@EmbeddedId`, `Persistable<ID>`
- Spring Data repository interfaces
- Custom repository implementations (batch operations, native queries)
- Liquibase XML changelogs (`{TICKET}-{description}.xml`)
- Repository Spock tests

**Deliverables**:
- `src/main/java/.../domain/entity/` - Entity classes
- `src/main/java/.../infrastructure/repository/` - Repository interfaces and implementations
- `src/main/resources/db/changelog/` - Liquibase changelogs
- `src/test/groovy/.../infrastructure/repository/` - Repository tests

**When to skip**: Story doesn't introduce new entities (pure API changes, refactoring existing code without schema changes)

---

### Phase 2: Integration Clients

**Agent**: `.claude/agents/integration-client-developer.md`

**Creates**:
- REST client classes with resilience patterns (CircuitBreaker, Retry)
- Client configuration classes with connection pooling and timeouts
- Client properties classes (`@ConfigurationProperties`)
- Request/Response DTOs with `@JsonIgnoreProperties`
- Client exception hierarchy (retryable vs non-retryable)
- Health indicators for external dependencies
- Client Spock tests using MockRestServiceServer

**Deliverables**:
- `src/main/java/.../infrastructure/client/{system}/` - Client classes
- `src/main/java/.../infrastructure/client/{system}/dto/` - Request/Response DTOs
- `src/main/java/.../infrastructure/client/{system}/exception/` - Client exceptions
- `src/test/groovy/.../infrastructure/client/{system}/` - Client tests

**Key Patterns**:
- URL placeholders with `{id}` syntax to avoid high-cardinality metrics
- Resilience4j for circuit breaker and retry logic
- MockRestServiceServer for request/response verification in tests
- Caching with `@Cacheable` for frequently accessed data

**When to skip**: Story doesn't require communication with external systems (pure internal processing, database-only operations)

---

### Phase 3: Domain Logic

**Agent**: `.claude/agents/domain-developer.md`

**Creates**:
- Facade classes (plain Java, no `@Service` annotation)
- Configuration classes with `@Bean` factory methods
- Metrics classes using Micrometer
- Exception hierarchy (Retryable vs Non-Retryable)
- Facade Spock tests with mocked dependencies

**Deliverables**:
- `src/main/java/.../domain/{feature}/` - Facade and service classes
- `src/main/java/.../domain/{feature}/` - Configuration classes
- `src/main/java/.../domain/{feature}/exceptions/` - Custom exceptions
- `src/test/groovy/.../domain/{feature}/` - Facade tests

**When to skip**: Story is pure CRUD with no business logic beyond data access

---

### Phase 4: REST API

**Agent**: `.claude/agents/rest-api-developer.md`

**Creates**:
- Controllers with OpenAPI/Swagger annotations
- Request/Response DTOs with Jakarta validation
- `.http` files for each endpoint (ArchUnit requirement)
- Controller Spock tests with MockMvc

**Deliverables**:
- `src/main/java/.../api/controller/` - Controller classes
- `src/main/java/.../api/controller/dto/` - DTOs
- `http/{ControllerName}/` - HTTP test files
- `src/test/groovy/.../api/controller/` - Controller tests

**When to skip**: Backend-only feature (scheduled job, Pub/Sub listener, background processor)

---

### Phase 5: Integration & Testing

**No specialized agent** - Manual verification and integration testing

**Activities**:
1. Run `./gradlew test` - Verify all Spock tests pass
2. Run `./gradlew check` - Verify ArchUnit compliance
3. Manual testing with `.http` files in IntelliJ IDEA
4. Fix any integration issues discovered
5. Update documentation if needed

**ArchUnit Rules Verified**:
- Every REST endpoint has corresponding `.http` file
- No field injection (constructor injection only)
- No deprecated API usage
- No standard streams access (`System.out`, `System.err`)
- No Java Util Logging (use SLF4J)
- No Joda Time (use `java.time` API)

---

### Phase 6: Code Review & Quality Gate

**Agent**: `.claude/agents/code-reviewer.md`

**Purpose**: Automated review of all code changes to catch architecture violations, design issues, and code quality problems before committing.

**When to Run**:
- After completing implementation phases (recommended)
- Before creating commits or pull requests (mandatory)
- After each individual phase for early feedback (optional)
- When manually requested: "Review my changes"

**What It Reviews**:
- **Architecture compliance**: Facade encapsulation, module boundaries, hexagonal patterns
- **Core principles**: KISS, YAGNI, SRP, DRY adherence
- **Domain patterns**: Rich entities, Tell Don't Ask, Law of Demeter
- **Persistence layer**: JPA annotations, Liquibase migrations, repository patterns
- **REST API**: Controller design, HTTP test files, OpenAPI compliance
- **Testing quality**: Coverage, meaningful tests, BDD structure
- **Code style**: Naming conventions, injection patterns, anti-patterns

**Review Process**:
1. Identifies modified files using `git diff`
2. Loads applicable rules from documentation
3. Analyzes each file against checklist
4. Categorizes issues (Critical 🔴, Moderate ⚠️, Minor 💡)
5. Provides actionable feedback with references

**Output Example**:
```markdown
## 🔍 Code Review Results

### Summary
- Files Reviewed: 5 files
- Issues Found: 3 issues (1 critical, 1 moderate, 1 minor)
- Overall Status: ⚠️ NEEDS IMPROVEMENT

### Critical Issues (Must Fix) 🔴
1. [Architecture] `ReserveOrchestrator.java:23`
   - Problem: Direct dependency on LimitRepository
   - Fix: Use LimitFacade instead
   - Reference: .claude/agents/domain-developer.md

### Positive Observations ✅
- Excellent facade encapsulation in CounterFacade
- Comprehensive test coverage with edge cases
```

**After Review**:
The agent offers to:
1. Fix all critical issues automatically
2. Fix specific issues you choose
3. Provide the report for manual fixes

**Integration with Workflow**:
- Run after Phase 4 (REST API) to review complete feature
- Run after individual phases for incremental feedback
- Always run before committing changes

## Quick Start Example

```bash
# 1. Create specification for Order Management feature
/spec UCRM "Order Management System"
# Creates: docs/specifications/SPEC-UCRM-001-order-management.md

# 2. Break into stories
/story UCRM-001 "Create new order via REST API"
# Creates: docs/stories/UCRM-001-create-order.md

# 3. Generate implementation plan
/plan UCRM-001
# Creates: docs/plans/PLAN-UCRM-001-create-order.md

# 4. Implement phase by phase
/implement Phase 1
# Persistence agent creates:
# - Order entity
# - OrderRepository
# - Liquibase changelog
# - Repository tests

/implement Phase 2
# Integration client agent creates:
# - InventoryServiceClient (to check stock)
# - InventoryServiceClientConfiguration
# - InventoryServiceClientProperties
# - Request/Response DTOs
# - Client tests with MockRestServiceServer

/implement Phase 3
# Domain agent creates:
# - OrderFacade
# - OrderConfiguration
# - OrderMetrics
# - Exception classes
# - Facade tests

/implement Phase 4
# REST API agent creates:
# - OrderController
# - CreateOrderRequest/OrderResponse DTOs
# - .http files
# - Controller tests

/implement Phase 5
# Manual integration testing:
# - Run ./gradlew test
# - Run ./gradlew check
# - Test with .http files
# - Fix any issues

# 5. Code review (automated quality gate)
/review
# OR for quick pre-commit check:
# /review quick

# Code Review Agent:
# - Analyzes all modified files
# - Checks architecture patterns (facade encapsulation!)
# - Verifies coding standards
# - Provides actionable feedback
# - Offers to fix critical issues

# 6. Verify and commit
./gradlew check
git add .
git commit -m "feat(UCRM-001): add order creation API with persistence and domain logic"
git push
```

## Phase Sequencing Strategy

**Why Inside-Out (Persistence → Clients → Domain → API)?**

This workflow follows an **inside-out** approach, starting from the data layer and building up:

### Advantages:

1. **Data Model First**: Database schema and entities are established before business logic, ensuring consistency
2. **External Dependencies Ready**: Integration clients are available before domain logic needs them
3. **Clear Dependencies**: Each layer depends only on the layers below it:
   - Domain Logic depends on Persistence AND Integration Clients
   - REST API depends on Domain Logic
4. **Test Isolation**: Each layer can be tested independently:
   - Repository tests use Testcontainers (real database)
   - Client tests use MockRestServiceServer (simulated HTTP)
   - Facade tests use mocked repositories and clients (fast unit tests)
   - Controller tests use MockMvc (Spring MVC integration)
5. **Agent Focus**: Each agent works with stable interfaces from the previous phase

### Layer Dependency Graph:

```
┌─────────────────────┐
│   REST API Layer    │  Controllers, DTOs
│   (Phase 4)         │  depends on ↓
└─────────────────────┘
          ↓
┌─────────────────────┐
│   Domain Layer      │  Facades, Services
│   (Phase 3)         │  depends on ↓
└─────────────────────┘
          ↓
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌─────────────────┐    ┌────────────────────┐ │
│  │  Persistence    │    │ Integration        │ │
│  │  Layer          │    │ Clients Layer      │ │
│  │  (Phase 1)      │    │ (Phase 2)          │ │
│  │                 │    │                    │ │
│  │  Entities,      │    │ REST Clients,      │ │
│  │  Repositories   │    │ Resilience         │ │
│  └─────────────────┘    └────────────────────┘ │
│                                                 │
│              INFRASTRUCTURE LAYER               │
└─────────────────────────────────────────────────┘
```

## Testing Strategy

This workflow follows a **test-after** approach, matching the existing codebase patterns:

### Test-After Workflow:
1. Implement the code first, following established patterns
2. Write Spock tests after implementation to verify behavior
3. Focus on meaningful test scenarios derived from BDD acceptance criteria

### Testing by Layer (Unit/Integration Tests):

| Layer | Test Type | Framework | Approach |
|-------|-----------|-----------|----------|
| Repository | Integration | Spock + Testcontainers | Test against real PostgreSQL database |
| Integration Client | Unit | Spock + MockRestServiceServer | Verify requests, simulate responses |
| Domain | Unit | Spock + Mocks | Test business logic with mocked dependencies |
| Controller | Integration | Spock + MockMvc | Test REST API with mocked facades |

### Component Tests (End-to-End)

Component tests verify the entire application in a production-like environment using TestContainers. These are separate from unit tests and provide confidence that all layers work together correctly.

| Aspect | Unit Tests | Component Tests |
|--------|-----------|-----------------|
| Scope | Single layer | Full application |
| Database | Mocked or Testcontainers | Real PostgreSQL in container |
| External Services | Mocked | WireMock in container |
| Application | Not running | Running in Docker container |
| Speed | Fast (milliseconds) | Slower (seconds per test) |
| When to run | Every build | Before release, CI pipeline |

See [Component Testing](#component-testing) section below for detailed agent usage.

### What Makes a Test Meaningful?

| Meaningful | Not Meaningful |
|------------|----------------|
| Tests that a user can create an order | Tests that a method was called |
| Tests that invalid input returns 400 | Tests that a mock was invoked |
| Tests that data persists to database | Tests that a setter sets a value |
| Tests that concurrent requests don't conflict | Tests that a constructor constructs |

## Flexibility: Skipping Phases

Not every story requires all implementation phases. The `/plan` command will identify which phases to skip based on the story's nature:

### When to Skip Phase 1 (Persistence):
- Story doesn't introduce new entities
- Pure API changes (adding endpoint to existing data)
- Refactoring existing code without schema changes
- UI-only changes or client-side logic

### When to Skip Phase 2 (Integration Clients):
- Story doesn't require external system communication
- Pure internal processing using existing data
- Database-only operations
- Modifying existing clients (not creating new ones)

### When to Skip Phase 3 (Domain Logic):
- Story is pure CRUD with no business logic
- Simple data pass-through to external service
- Configuration changes only
- Infrastructure updates (Docker, CI/CD)

### When to Skip Phase 4 (REST API):
- Backend-only feature (scheduled job, cron task)
- Pub/Sub message listener
- Background processor or worker
- Database migration only

### Examples:

**Full Stack Story** (All Phases):
- _"Create order with validation and inventory check from warehouse service"_
- Phase 1: Order entity, OrderRepository
- Phase 2: WarehouseServiceClient for inventory check
- Phase 3: OrderFacade with inventory validation logic
- Phase 4: OrderController with POST endpoint

**API-Only Story** (Skip Phases 1 & 2):
- _"Add pagination to existing customer list API"_
- ~~Phase 1~~: (Skip - no new entities)
- ~~Phase 2~~: (Skip - no external calls)
- Phase 3: Update CustomerFacade with pagination
- Phase 4: Update CustomerController with pagination params

**External Integration Story** (Skip Phases 1 & 4):
- _"Fetch customer credit score from external rating service"_
- ~~Phase 1~~: (Skip - no new entities, store in existing Customer)
- Phase 2: CreditRatingClient with resilience patterns
- Phase 3: CustomerFacade enrichment logic
- ~~Phase 4~~: (Skip - triggered by existing endpoint or scheduler)

**Backend-Only Story** (Skip Phases 2 & 4):
- _"Add daily cleanup job for expired sessions"_
- Phase 1: SessionRepository cleanup method
- ~~Phase 2~~: (Skip - no external calls)
- Phase 3: CleanupFacade with scheduling logic
- ~~Phase 4~~: (Skip - no REST API needed)

## Integration with Existing Patterns

### CLAUDE.md Guidelines
This workflow complements the existing `CLAUDE.md` project guidelines:
- Follows KISS, YAGNI, SRP, DRY principles
- Uses ticket-based naming (UCRM-XXXXX)
- Adheres to package structure conventions
- Respects ArchUnit rules

### ArchUnit Enforcement
All implementation phases must comply with ArchUnit rules:
- Every REST endpoint has `.http` file (validated by `ArchitectureTest.groovy`)
- No field injection - constructor injection only
- No deprecated APIs
- No standard streams
- SLF4J logging only

Run `./gradlew check` in Phase 5 to verify compliance.

### Liquibase Conventions
Database changes follow existing patterns:
- Master changelog: `db.changelog-master.xml`
- Individual changelogs: `{TICKET}-{description}.xml`
- Include new changelogs at end of master
- Use `logicalFilePath` for refactoring safety

### Spock Testing
All tests use Spock framework with Groovy:
- `given/when/then` structure
- Mocking with `Mock()` syntax
- Right-arrow operators for stubbing (`>>`)
- Data tables for parameterized tests

## Component Testing

Component tests provide end-to-end verification of the application using TestContainers. Unlike unit tests that run during each phase, component tests are managed separately and verify the complete processing pipeline.

### Two-Agent Approach

Component testing uses two specialized agents with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPONENT TEST INFRASTRUCTURE                       │
│  Agent: .claude/agents/component-test-infrastructure.md         │
│                                                                 │
│  Run ONCE per project to set up:                                │
│  • Gradle configuration (source sets, dependencies)            │
│  • ContainersSetup (PostgreSQL, WireMock, GCP emulators)       │
│  • Base classes (BDDScenario, BaseComponentTest, TestHelpers)  │
│  • Test clients (GcsTestClient, PubSubTestClient)              │
│  • Test resources and data builders                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              COMPONENT TEST DEVELOPMENT                          │
│  Agent: .claude/agents/component-test-developer.md              │
│                                                                 │
│  Run for EACH feature/story to create:                          │
│  • Test classes extending BaseComponentTest                     │
│  • BDD scenarios (given/when/then)                             │
│  • WireMock stubs for external services                        │
│  • Database verifications via TestHelpers                       │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Component Tests

| Scenario | Use Component Tests? |
|----------|---------------------|
| Complex multi-step workflows | ✅ Yes |
| External service integrations | ✅ Yes |
| Event-driven processing (Pub/Sub) | ✅ Yes |
| Scheduler-based processing | ✅ Yes |
| Simple CRUD operations | ❌ Unit tests sufficient |
| Single-layer changes | ❌ Unit tests sufficient |

### Infrastructure Setup (One-Time)

Use the **Component Test Infrastructure Agent** when setting up a new project or adding new infrastructure:

```bash
# Set up component test infrastructure
# Agent creates: ContainersSetup, BaseComponentTest, TestHelpers,
#                GcsTestClient, PubSubTestClient, Gradle config

# This is done ONCE per project, not per story
```

**Deliverables**:
- `src/componentTest/java/.../ContainersSetup.java`
- `src/componentTest/java/.../BaseComponentTest.java`
- `src/componentTest/java/.../BDDScenario.java`
- `src/componentTest/java/.../TestHelpers.java`
- `src/componentTest/java/.../GcsTestClient.java`
- `src/componentTest/java/.../PubSubTestClient.java`
- `src/componentTest/java/.../WireMockService.java`
- `src/componentTest/java/.../TestDataBuilders.java`
- `src/componentTest/resources/logback-test.xml`
- `src/componentTest/resources/test/files/*.csv`
- `build.gradle.kts` updates

### Test Development (Per Feature)

Use the **Component Test Developer Agent** when writing tests for a feature:

```bash
# Write component tests for file processing feature
# Agent creates: FileIngestionComponentTest, EnrichmentComponentTest,
#                InvestmentsAppStubs, HiLoStubs

# Run for each major feature or story
```

**Deliverables**:
- `src/componentTest/java/.../fileprocessing/FileIngestionComponentTest.java`
- `src/componentTest/java/.../fileprocessing/EnrichmentComponentTest.java`
- `src/componentTest/java/.../fileprocessing/BookingComponentTest.java`
- `src/componentTest/java/.../fileprocessing/stubs/InvestmentsAppStubs.java`
- `src/componentTest/java/.../fileprocessing/stubs/HiLoStubs.java`

### BDD Pattern Enforcement

All component tests MUST use the fluent BDD API:

```java
@Test
void shouldProcessFileSuccessfully() {
    given("valid CSV file uploaded to GCS", () -> {
        gcsTestClient.uploadFile("test/files/valid-transactions.csv", "GL_20251017.csv");
    });

    when("Pub/Sub file notification is published", () -> {
        pubSubTestClient.publishFileNotification("GL_20251017.csv");
    });

    then("transactions are ingested with PENDING status", () -> {
        await()
            .atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                var count = testHelpers.countRows("transaction", "status = ?", "PENDING");
                assertThat(count).isEqualTo(4);
            });
    });
}
```

### Running Component Tests

```bash
# Build application image first
./gradlew bootJar

# Run component tests
./gradlew componentTest

# Run with specific test class
./gradlew componentTest --tests "FileIngestionComponentTest"
```

## File Organization

After following this workflow, your repository will have:

```
{project-root}/
├── docs/
│   ├── specifications/           # High-level PRDs
│   │   └── SPEC-{PROJECT}-{NUM}-{name}.md
│   ├── stories/                  # User stories with BDD
│   │   └── {PROJECT}-{NUM}-{short-desc}.md
│   ├── plans/                    # Implementation plans
│   │   └── PLAN-{PROJECT}-{NUM}-{feature}.md
│   └── architecture/             # ADRs (existing)
│       └── ADR0XX-*.md
│
├── src/
│   ├── main/
│   │   ├── java/com/{company}/{app}/
│   │   │   ├── api/controller/           # Phase 4: REST API
│   │   │   │   ├── dto/
│   │   │   │   └── *Controller.java
│   │   │   ├── domain/                   # Phase 3: Domain Logic
│   │   │   │   ├── entity/
│   │   │   │   └── {feature}/
│   │   │   │       ├── *Facade.java
│   │   │   │       ├── *Configuration.java
│   │   │   │       ├── *Metrics.java
│   │   │   │       └── exceptions/
│   │   │   └── infrastructure/           # Phase 1 & 2
│   │   │       ├── repository/           # Phase 1: Persistence
│   │   │       │   └── *Repository.java
│   │   │       └── client/               # Phase 2: Integration Clients
│   │   │           └── {external-system}/
│   │   │               ├── *Client.java
│   │   │               ├── *ClientConfiguration.java
│   │   │               ├── *ClientProperties.java
│   │   │               ├── *HealthIndicator.java
│   │   │               ├── dto/
│   │   │               │   ├── *Request.java
│   │   │               │   └── *Response.java
│   │   │               └── exception/
│   │   │                   └── *ClientException.java
│   │   └── resources/
│   │       └── db/changelog/
│   │           ├── db.changelog-master.xml
│   │           └── {TICKET}-*.xml
│   │
│   ├── test/groovy/...                   # Spock unit tests (all phases)
│   │
│   └── componentTest/                    # Component tests (E2E)
│       ├── java/com/{company}/{app}/
│       │   ├── BDDScenario.java          # Fluent API interface
│       │   ├── BaseComponentTest.java    # Base test class
│       │   ├── ContainersSetup.java      # TestContainers config
│       │   ├── TestHelpers.java          # SQL utilities
│       │   ├── GcsTestClient.java        # GCS emulator client
│       │   ├── PubSubTestClient.java     # Pub/Sub emulator client
│       │   ├── WireMockService.java      # WireMock base class
│       │   ├── TestDataBuilders.java     # Entity factories
│       │   └── {feature}/                # Feature test classes
│       │       ├── *ComponentTest.java
│       │       └── stubs/
│       │           └── *Stubs.java
│       └── resources/
│           ├── logback-test.xml
│           ├── junit-platform.properties
│           └── test/files/
│               └── *.csv
│
├── http/                                 # HTTP test files
│   └── {Controller}/
│       └── {methodName}.http
│
└── .claude/
    ├── commands/                         # Slash commands
    │   ├── spec.md
    │   ├── story.md
    │   ├── plan.md
    │   └── implement.md
    └── agents/                           # Implementation agents
        ├── persistence-developer.md
        ├── integration-client-developer.md
        ├── domain-developer.md
        ├── rest-api-developer.md
        ├── code-reviewer.md              # Quality assurance agent
        ├── component-test-infrastructure.md
        └── component-test-developer.md
```

## Using External Specifications

The workflow is **flexible** and works with specifications created outside the `/spec` command:

### Supported Scenarios

✅ **Manually written specifications**
- Place in `docs/specifications/` directory
- Name with `SPEC-{PROJECT}-{NUM}-{name}.md` or `{PROJECT}-{name}.md`
- Any markdown format works (not bound to `/spec` template)

✅ **Imported specifications**
- Confluence exports, Google Docs converted to markdown
- Technical design documents
- Architecture Decision Records (ADRs)

✅ **No specification at all**
- Start directly with `/story` command
- Specify "Standalone" when prompted
- Claude will ask for necessary context

### How Commands Handle External Specs

| Command | Behavior with External Spec |
|---------|---------------------------|
| `/story` | Searches `docs/specifications/` for `SPEC-{PROJECT}-*.md` or `{PROJECT}-*.md`. Extracts context flexibly (not strict template). Falls back gracefully if not found. |
| `/plan` | Only needs the story file (`docs/stories/{PROJECT}-{NUM}-*.md`). Spec not required. |
| `/implement` | Only needs the plan file. Spec not required. |

### Example: Using External Spec

```bash
# 1. You have an external spec (any format)
# Place it at: docs/specifications/UCRM-order-system-design.md

# 2. Create story - will find and read the spec
/story UCRM-001 "Create order API"
# → Searches docs/specifications/ for UCRM-*.md
# → Reads your external spec
# → Extracts relevant context
# → Creates story with BDD scenarios

# 3. Continue normal workflow
/plan UCRM-001
/implement Phase 1
```

### Best Practices for External Specs

**Recommended sections** (for best context extraction):
- Overview/Summary
- Requirements or Features
- Data Models or Entities
- API Contracts or Endpoints

**Not required but helpful**:
- Use markdown headers (`##`, `###`)
- Include code blocks for data models
- Link to related ADRs

**What works even without structure**:
- Plain text descriptions
- Bullet point lists
- Any markdown content

Claude will adapt to whatever format you provide.

## Best Practices

### Specification Phase:
- Reference existing ADR files for architecture decisions
- Include specific examples of data models and API contracts
- Define success criteria upfront
- Consider security, performance, and scalability

### Story Decomposition:
- Keep stories small and independently deliverable
- Write comprehensive BDD acceptance criteria
- Include error cases and edge cases
- Link back to parent specification

### Implementation Planning:
- Identify which phases can be skipped early
- Break tasks into checkboxes for progress tracking
- Define clear completion criteria per phase
- Include meaningful test scenarios from BDD criteria

### Implementation Execution:
- Complete one phase fully before moving to the next
- Update plan checkboxes as you progress
- Run tests after each phase
- Keep commits focused on single phase

### Verification:
- Run `./gradlew test` to verify all tests pass
- Run `./gradlew check` to verify ArchUnit compliance
- Manually test with `.http` files
- Review code against BDD acceptance criteria
- Ensure all checkboxes in plan are marked complete

## Related Documentation

- [CLAUDE.md](./CLAUDE.md) - Project guidelines and conventions
- [ArchitectureTest.groovy](src/test/groovy/.../ArchitectureTest.groovy) - ArchUnit rules
- [ADR files](docs/architecture/) - Architecture Decision Records

### Implementation Agents
- [Persistence Agent](.claude/agents/persistence-developer.md) - Database layer patterns
- [Integration Client Agent](.claude/agents/integration-client-developer.md) - External system client patterns
- [Domain Logic Agent](.claude/agents/domain-developer.md) - Business logic patterns
- [REST API Agent](.claude/agents/rest-api-developer.md) - REST API implementation patterns

### Quality Assurance Agents
- [Code Review Agent](.claude/agents/code-reviewer.md) - Automated code review, architecture compliance

### Component Test Agents
- [Component Test Infrastructure Agent](.claude/agents/component-test-infrastructure.md) - TestContainers setup, base classes
- [Component Test Developer Agent](.claude/agents/component-test-developer.md) - BDD test scenarios, WireMock stubs

---

## Summary

This workflow provides a structured approach to AI-assisted Spring Boot microservices development:

1. **Specification** → Define what to build with PRD-style document
2. **Stories** → Break down into implementable units with BDD criteria
3. **Planning** → Generate phased implementation plan with agent assignments
4. **Implementation** → Execute phases using specialized agents
5. **Verification** → Test, validate with ArchUnit, and deliver
6. **Component Testing** → End-to-end verification with TestContainers (optional)

By following this workflow, you ensure:
- Clear traceability from specification to code
- Consistent implementation patterns across the team
- Comprehensive testing at each layer (unit + component)
- ArchUnit compliance throughout
- Independent, parallelizable work units (stories)

### Agent Summary

| Agent | Purpose | When to Use |
|-------|---------|-------------|
| Persistence Developer | Entities, repositories, Liquibase | New database tables/columns |
| Integration Client Developer | REST clients, resilience patterns | External system communication |
| Domain Logic Developer | Facades, services, metrics | Business logic |
| REST API Developer | Controllers, DTOs, .http files | API endpoints |
| Code Review Agent | Architecture compliance, quality checks | After implementation, before commits |
| Component Test Infrastructure | TestContainers, base classes | Once per project |
| Component Test Developer | BDD test scenarios, stubs | Per feature/story |
