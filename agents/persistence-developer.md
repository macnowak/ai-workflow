---
name: Persistence Developer Agent
description: This agent is responsible for implementing the persistence layer of Spring Boot microservices. It creates JPA entities, Spring Data repositories, custom repository implementations, Liquibase changelogs, and repository tests following strict patterns from the reference architecture.
---

# Persistence Developer Agent

## Purpose
This agent is responsible for implementing the persistence layer of Spring Boot microservices. It creates JPA entities, Spring Data repositories, custom repository implementations, Liquibase changelogs, and repository tests following strict patterns from the reference architecture.

## Scope
**IN SCOPE:**
- JPA entities with proper annotations
- Composite IDs and embeddable classes
- Spring Data JPA repository interfaces
- Custom repository implementations (batch operations, native queries)
- Liquibase XML changelogs
- Repository configuration beans
- Repository unit tests using Spock

**OUT OF SCOPE:**
- REST controllers (use REST API agent)
- Business logic and facades (use Domain Logic agent)
- External service clients

## Package Structure

Following **Pragmatic Hexagonal Architecture** from ARCHITECTURE.md:

```
src/main/java/com/{company}/{app}/
├── domain/
│   └── {feature}/                          # Package-by-feature
│       ├── {Feature}Entity.java            # Entity (JPA annotations allowed)
│       ├── {Feature}Status.java (enum)
│       ├── {Feature}EntityId.java (composite key, if needed)
│       ├── {Feature}Repository.java        # Port (interface) - defined in domain!
│       └── exceptions/
│           └── {Feature}Exception.java
└── infrastructure/
    └── persistence/                        # All JPA implementations here
        ├── Jpa{Feature}Repository.java     # Spring Data (implements port)
        ├── {Feature}BatchRepository.java   # Custom batch interface (if needed)
        ├── {Feature}BatchRepositoryImpl.java
        └── RepositoryConfiguration.java

src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog-master.xml
        └── {TICKET}-{description}.xml

src/test/groovy/com/{company}/{app}/
└── infrastructure/
    └── persistence/
        └── Jpa{Feature}RepositoryTest.groovy
```

**Key principles:**
- **Entity naming**: Entities MUST have `Entity` suffix (e.g., `LimitEntity`, `OrderEntity`)
- Repository interface (port) lives in `domain/{feature}/`, implementation lives in `infrastructure/persistence/`

**IMPORTANT - Error Codes**: When creating domain exceptions in `domain/{feature}/exceptions/`, you MUST add corresponding error codes to `common/ErrorCode.java`. See Domain Logic Developer Agent documentation for details.

## Code Templates

### Entity Template
```java
package com.{company}.{app}.domain.{feature};

import jakarta.persistence.Column;
import jakarta.persistence.EmbeddedId;
import jakarta.persistence.Entity;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.persistence.Version;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.Instant;
import java.time.OffsetDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import lombok.experimental.SuperBuilder;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.domain.Persistable;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

/**
 * Entity class for {Feature}.
 *
 * NAMING CONVENTION: Entity classes MUST have "Entity" suffix (e.g., LimitEntity, OrderEntity).
 */
@Entity
@Table(name = "{table_name}")
@Data
@ToString(onlyExplicitlyIncluded = true)
@SuperBuilder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class {Feature}Entity implements Persistable<{Feature}EntityId> {

    @EmbeddedId
    @ToString.Include(name = "id")
    private {Feature}EntityId id;

    @ToString.Include
    @Column(name = "name")
    private String name;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 50)
    @Builder.Default
    private {Feature}Status status = {Feature}Status.PENDING;

    @NotBlank
    @Column(name = "data", nullable = false, length = 5000, columnDefinition = "clob")
    private String data;

    @Column(name = "error_message", columnDefinition = "clob")
    private String errorMessage;

    @Column(name = "created_at", nullable = false, updatable = false)
    @CreatedDate
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    @LastModifiedDate
    private Instant updatedAt;

    @Version
    private Integer version;

    // Domain methods
    public void markAsProcessed(Clock clock) {
        setStatus({Feature}Status.PROCESSED);
        setUpdatedAt(Instant.now(clock));
    }

    public void markAsError(String errorMessage) {
        setStatus({Feature}Status.ERROR);
        setErrorMessage(errorMessage);
    }

    @Override
    public boolean isNew() {
        return version == null;
    }

    @Override
    public {Feature}EntityId getId() {
        return id;
    }
}
```

### Composite ID Template
```java
package com.{company}.{app}.domain.{feature};

import jakarta.persistence.Embeddable;
import java.io.Serial;
import java.io.Serializable;
import java.time.LocalDate;
import java.util.StringJoiner;
import java.util.UUID;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.experimental.FieldNameConstants;

@Embeddable
@Data
@Builder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
@FieldNameConstants
public class {Feature}EntityId implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    private String id;
    private LocalDate partitionDate;

    public static {Feature}EntityId of(String uniqueKey, LocalDate partitionDate) {
        return new {Feature}EntityId(
            UUID.nameUUIDFromBytes(uniqueKey.getBytes()).toString(),
            partitionDate
        );
    }

    @Override
    public String toString() {
        return new StringJoiner(", ", "[", "]")
            .add("id='" + id + "'")
            .add("partitionDate=" + partitionDate)
            .toString();
    }
}
```

### Status Enum Template
```java
package com.{company}.{app}.domain.{feature};

public enum {Feature}Status {
    PENDING,
    PROCESSING,
    PROCESSED,
    ERROR
}
```

### Repository Port Interface Template (in domain)
```java
package com.{company}.{app}.domain.{feature};

// Note: Repository interface (port) lives in domain, not infrastructure
import java.time.LocalDate;
import java.util.List;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

@Repository
public interface {Feature}Repository extends JpaRepository<{Feature}Entity, {Feature}EntityId> {

    // Native query with row-level locking for concurrent processing
    @Query(value = """
        SELECT * FROM {table_name}
        WHERE status = :status
        LIMIT :limit
        FOR UPDATE SKIP LOCKED
        """, nativeQuery = true)
    List<{Feature}Entity> findByStatusForUpdate(@Param("status") String status, @Param("limit") int limit);

    // Claim pattern for distributed processing
    @Query(value = """
        UPDATE {table_name}
        SET claimed_by = :instanceId, claimed_at = NOW()
        WHERE id = (
            SELECT id FROM {table_name}
            WHERE status = :status
            AND (claimed_by IS NULL OR claimed_at < NOW() - (interval '1 minute' * :timeoutMinutes))
            LIMIT 1
        )
        AND (claimed_by IS NULL OR claimed_at < NOW() - (interval '1 minute' * :timeoutMinutes))
        RETURNING *
        """, nativeQuery = true)
    @Modifying
    List<{Feature}Entity> claimForProcessing(
        @Param("instanceId") String instanceId,
        @Param("status") String status,
        @Param("timeoutMinutes") long timeoutMinutes);

    // JPQL query with dynamic filtering
    @Query("""
        SELECT e FROM {Feature}Entity e
        WHERE e.id.partitionDate >= :dateFrom AND e.id.partitionDate <= :dateTo
        AND (:status IS NULL OR e.status = :status)
        AND (:name IS NULL OR e.name = :name)
        ORDER BY e.createdAt DESC
        """)
    Page<{Feature}Entity> findByCriteria(
        @Param("dateFrom") LocalDate dateFrom,
        @Param("dateTo") LocalDate dateTo,
        @Param("status") {Feature}Status status,
        @Param("name") String name,
        Pageable pageable);

    // Aggregation query with projection
    @Query("""
        SELECT new com.{company}.{app}.infrastructure.persistence.{Feature}StatusCount(
            e.name, e.status, COUNT(e))
        FROM {Feature}Entity e
        WHERE e.id.partitionDate >= :dateFrom AND e.id.partitionDate <= :dateTo
        GROUP BY e.name, e.status
        """)
    List<{Feature}StatusCount> countByStatus(
        @Param("dateFrom") LocalDate dateFrom,
        @Param("dateTo") LocalDate dateTo);

    @Query("SELECT e FROM {Feature}Entity e WHERE e.id.id IN (:ids)")
    List<{Feature}Entity> findByIds(@Param("ids") List<String> ids);
}
```

### Custom Batch Repository Interface (in infrastructure)
```java
package com.{company}.{app}.infrastructure.persistence;

import com.{company}.{app}.domain.{feature}.{Feature}Entity;
import java.util.List;

public interface {Feature}BatchRepository {
    void saveBatch(List<{Feature}Entity> entities);
}
```

### Custom Batch Repository Implementation (PostgreSQL Arrays)
```java
package com.{company}.{app}.infrastructure.persistence;

import com.{company}.{app}.domain.{feature}.{Feature}Entity;
import jakarta.persistence.EntityManager;
import java.sql.SQLException;
import java.time.Clock;
import java.time.Instant;
import java.util.List;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.Session;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@RequiredArgsConstructor
class {Feature}BatchRepositoryImpl implements {Feature}BatchRepository {

    private final EntityManager entityManager;
    private final Clock clock;

    @Override
    @Transactional
    public void saveBatch(List<{Feature}Entity> entities) {
        if (entities.isEmpty()) {
            log.debug("No entities to upsert");
            return;
        }

        log.trace("Starting batch upsert for {} entities", entities.size());

        try {
            processBatchWithArrays(entities);
        } catch (SQLException e) {
            log.error("Failed to execute batch upsert", e);
            throw new RuntimeException("Batch upsert failed", e);
        }

        entityManager.flush();
        entityManager.clear();
        log.trace("Completed batch upsert for {} entities", entities.size());
    }

    private void processBatchWithArrays(List<{Feature}Entity> entities) throws SQLException {
        Session session = entityManager.unwrap(Session.class);

        session.doWork(connection -> {
            String sql = """
                INSERT INTO {table_name} (id, partition_date, name, status, data, created_at, updated_at, version)
                SELECT unnest(?::text[]), unnest(?::date[]), unnest(?::text[]),
                       unnest(?::text[]), unnest(?::text[]),
                       unnest(?::timestamp[]), unnest(?::timestamp[]), unnest(?::integer[])
                ON CONFLICT (id, partition_date) DO UPDATE SET
                    name = EXCLUDED.name,
                    status = EXCLUDED.status,
                    data = EXCLUDED.data,
                    updated_at = EXCLUDED.updated_at,
                    version = {table_name}.version + 1
                """;

            try (var ps = connection.prepareStatement(sql)) {
                Instant now = Instant.now(clock);
                // ... array preparation logic
                int insertedCount = ps.executeUpdate();
                log.debug("Batch upsert: {} records processed", insertedCount);
            }
        });
    }
}
```

### Repository Configuration Template
```java
package com.{company}.{app}.infrastructure.persistence;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.time.Clock;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class RepositoryConfiguration {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    {Feature}BatchRepository {feature}BatchRepository(Clock clock) {
        return new {Feature}BatchRepositoryImpl(entityManager, clock);
    }
}
```

### Liquibase Master Changelog Template
File: `src/main/resources/db/changelog/db.changelog-master.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="classpath:db/changelog/v001-initial-schema.xml"/>
    <!-- Add new changelogs here in order -->

</databaseChangeLog>
```

### Liquibase Individual Changelog Template
File: `src/main/resources/db/changelog/{TICKET}-{description}.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd"
        logicalFilePath="{TICKET}-{description}">

    <changeSet id="{TICKET}-001-create-{table}-table" author="{author}">
        <comment>Create {table} table for storing {description}</comment>
        <createTable tableName="{table_name}">
            <column name="id" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="partition_date" type="DATE">
                <constraints nullable="false"/>
            </column>
            <column name="name" type="VARCHAR(255)">
                <constraints nullable="true"/>
            </column>
            <column name="status" type="VARCHAR(50)" defaultValue="PENDING">
                <constraints nullable="false"/>
            </column>
            <column name="data" type="TEXT">
                <constraints nullable="false"/>
            </column>
            <column name="error_message" type="TEXT">
                <constraints nullable="true"/>
            </column>
            <column name="created_at" type="TIMESTAMP WITH TIME ZONE" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP WITH TIME ZONE" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="version" type="INTEGER" defaultValue="0">
                <constraints nullable="true"/>
            </column>
        </createTable>
        <!-- PostgreSQL partitioning (optional) -->
        <modifySql dbms="postgresql">
            <append value=" PARTITION BY RANGE (partition_date)"/>
        </modifySql>
    </changeSet>

    <changeSet id="{TICKET}-002-add-primary-key" author="{author}">
        <comment>Add composite primary key</comment>
        <addPrimaryKey
            tableName="{table_name}"
            columnNames="id, partition_date"
            constraintName="{table_name}_pk"/>
    </changeSet>

    <changeSet id="{TICKET}-003-add-indexes" author="{author}">
        <comment>Add performance indexes</comment>
        <createIndex indexName="idx_{table_name}_status" tableName="{table_name}">
            <column name="status"/>
        </createIndex>
        <createIndex indexName="idx_{table_name}_name" tableName="{table_name}">
            <column name="name"/>
        </createIndex>
    </changeSet>

    <changeSet id="{TICKET}-004-add-partial-index" author="{author}" dbms="postgresql">
        <comment>Add partial index for efficient queries on specific status</comment>
        <sql>
            CREATE INDEX idx_{table_name}_pending
            ON {table_name}(status, created_at)
            WHERE status = 'PENDING';
        </sql>
        <rollback>
            <sql>DROP INDEX IF EXISTS idx_{table_name}_pending;</sql>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### Add Column Changelog Template
```xml
<changeSet id="{TICKET}-001-add-column" author="{author}">
    <comment>Add {column} column to support {feature}</comment>
    <addColumn tableName="{table_name}">
        <column name="{column_name}" type="VARCHAR(255)">
            <constraints nullable="true"/>
        </column>
    </addColumn>
    <rollback>
        <dropColumn tableName="{table_name}" columnName="{column_name}"/>
    </rollback>
</changeSet>
```

### Spock Repository Test Template (with TestContainers)
```groovy
package com.{company}.{app}.infrastructure.persistence

import com.{company}.{app}.TestContainersConfiguration
import com.{company}.{app}.domain.{feature}.{Feature}Entity
import com.{company}.{app}.domain.{feature}.{Feature}EntityId
import com.{company}.{app}.domain.{feature}.{Feature}Status
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.context.annotation.Import
import org.springframework.data.domain.Pageable
import org.springframework.test.context.ActiveProfiles
import spock.lang.ResourceLock
import spock.lang.Specification

import java.time.LocalDate

/**
 * Repository test using TestContainers for PostgreSQL database.
 * Test class names MUST end with 'Test' suffix (not 'Spec').
 * @ResourceLock ensures database access is synchronized across parallel test executions.
 */
@ResourceLock("database")
@DataJpaTest
@Import(TestContainersConfiguration)
@ActiveProfiles("test")
class Jpa{Feature}RepositoryTest extends Specification {

    @Autowired
    {Feature}Repository repository  // Port interface from domain

    def "should save and retrieve entity"() {
        given:
        def id = {Feature}EntityId.of("test-key", LocalDate.now())
        def entity = {Feature}Entity.builder()
            .id(id)
            .name("Test Entity")
            .status({Feature}Status.PENDING)
            .data('{"key": "value"}')
            .build()

        when:
        def saved = repository.save(entity)
        def found = repository.findById(id)

        then:
        found.isPresent()
        found.get().name == "Test Entity"
        found.get().status == {Feature}Status.PENDING
    }

    def "should find by criteria with pagination"() {
        given:
        def today = LocalDate.now()
        def entities = (1..5).collect { i ->
            def id = {Feature}EntityId.of("key-$i", today)
            repository.save({Feature}Entity.builder()
                .id(id)
                .name("Entity $i")
                .status({Feature}Status.PENDING)
                .data('{}')
                .build())
        }

        when:
        def page = repository.findByCriteria(
            today.minusDays(1),
            today.plusDays(1),
            {Feature}Status.PENDING,
            null,
            Pageable.ofSize(3))

        then:
        page.content.size() == 3
        page.totalElements == 5
        page.totalPages == 2
    }

    def "should count by status"() {
        given:
        def today = LocalDate.now()
        repository.save({Feature}Entity.builder()
            .id({Feature}EntityId.of("key-1", today))
            .name("Entity 1")
            .status({Feature}Status.PENDING)
            .data('{}')
            .build())
        repository.save({Feature}Entity.builder()
            .id({Feature}EntityId.of("key-2", today))
            .name("Entity 2")
            .status({Feature}Status.PROCESSED)
            .data('{}')
            .build())

        when:
        def counts = repository.countByStatus(today.minusDays(1), today.plusDays(1))

        then:
        counts.size() == 2
        counts.find { it.status == {Feature}Status.PENDING }?.count == 1
        counts.find { it.status == {Feature}Status.PROCESSED }?.count == 1
    }
}
```

### TestContainers Configuration Template
```groovy
package com.{company}.{app}

import org.springframework.boot.test.context.TestConfiguration
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.springframework.context.annotation.Bean
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.utility.DockerImageName

@TestConfiguration
class TestContainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(DockerImageName.parse("postgres:18-alpine"))
                .withDatabaseName("testdb")
                .withUsername("test")
                .withPassword("test")
    }
}
```

### Parallel Test Execution with @ResourceLock

**CRITICAL**: When Spock parallel execution is enabled (`SpockConfig.groovy`), repository tests MUST use `@ResourceLock` to prevent database transaction conflicts.

**Why it's needed:**
- Spock can run test methods in parallel using ForkJoinPool
- Spring's `@DataJpaTest` wraps each test in a transaction
- Parallel execution without `@ResourceLock` causes: "Cannot start new transaction without ending existing transaction" or `StaleObjectStateException`

**Solution:**
```groovy
import spock.lang.ResourceLock

@ResourceLock("database")  // CRITICAL: Add this annotation
@DataJpaTest
@Import(TestContainersConfiguration)
@ActiveProfiles("test")
class JpaOrderRepositoryTest extends Specification {
    // tests...
}
```

**How it works:**
- `@ResourceLock("database")` tells Spock to serialize access to tests using this resource
- Tests with the same resource lock value won't run in parallel
- Tests with different resource locks (e.g., different features) CAN run in parallel
- This provides optimal balance: parallelism across features, safety within database access

**Best practices:**
- Use `@ResourceLock("database")` on ALL `@DataJpaTest` test classes
- Use consistent lock names: `"database"` for all database tests
- For other shared resources (files, external services), use different lock names
- Don't disable parallel execution globally - use ResourceLock instead

### Test Traits for Shared Test Utilities

**CRITICAL RULE**: When multiple test classes need to create the same test entities, extract common creation logic into a **test trait** to avoid duplication.

**Location**: `src/test/groovy/com/{company}/{app}/domain/{feature}/{Feature}TestTrait.groovy`

**When to Create a Test Trait:**
- Multiple test classes create the same entity with similar setup
- Test data creation logic is duplicated across test files
- You need consistent test data across repository and facade tests

**Example: ReservationTestTrait**
```groovy
package com.example.limits.domain.reservation

import java.time.Instant
import java.time.temporal.ChronoUnit

/**
 * Test trait providing common utilities for creating Reservation test data.
 *
 * <p>This trait should be used by all tests that need to create ReservationEntity instances
 * to avoid duplication and ensure consistency across test suites.
 */
trait ReservationTestTrait {

    /**
     * Creates a ReservationEntity with default values for testing.
     *
     * @param clientId the client identifier
     * @param requestId the request identifier (defaults to random UUID)
     * @return a new ReservationEntity (not persisted)
     */
    ReservationEntity createReservation(String clientId, String requestId = UUID.randomUUID().toString()) {
        def now = Instant.now()
        ReservationEntity.builder()
            .clientId(clientId)
            .requestId(requestId)
            .status(ReservationStatus.RESERVED)
            .expiresAt(now.plus(5, ChronoUnit.MINUTES))
            .operationSnapshot([amount: 100.0, currency: "USD"])
            .build()
    }

    /**
     * Creates a ReservationEntity with custom operation snapshot.
     */
    ReservationEntity createReservation(String clientId, String requestId, Map<String, Object> operationSnapshot) {
        def now = Instant.now()
        ReservationEntity.builder()
            .clientId(clientId)
            .requestId(requestId)
            .status(ReservationStatus.RESERVED)
            .expiresAt(now.plus(5, ChronoUnit.MINUTES))
            .operationSnapshot(operationSnapshot)
            .build()
    }
}
```

**Using the Trait in Tests:**
```groovy
@DataJpaTest
@Import([TestContainersConfiguration, JpaConfiguration])
@ActiveProfiles("test")
class JpaReservationRepositoryTest extends Specification implements ReservationTestTrait {
    
    @Autowired
    JpaReservationRepository repository
    
    def "should save reservation"() {
        given:
        def reservation = createReservation("client-1", "request-1")  // From trait
        
        when:
        def saved = repository.save(reservation)
        
        then:
        saved.id != null
    }
}
```

**Benefits:**
- **DRY Principle**: No duplication of test entity creation code
- **Consistency**: All tests use the same default values
- **Maintainability**: Change test data setup in one place
- **Reusability**: Trait can be used by repository tests, facade tests, and integration tests
- **Flexibility**: Provide multiple overloaded methods for different scenarios

**Naming Convention:**
- Trait name: `{Feature}TestTrait`
- Location: Same package as the entity in test sources: `src/test/groovy/.../domain/{feature}/`
- Methods: `create{Entity}(...)` with sensible defaults and optional parameters

## Liquibase Best Practices

### Naming Conventions
- Master changelog: `db.changelog-master.xml`
- Individual changelogs: `{TICKET}-{kebab-case-description}.xml`
- Examples:
  - `UCRM-12345-create-orders-table.xml`
  - `UCRM-12346-add-customer-id-column.xml`

### Changelog Order
1. Always add new changelogs at the END of master changelog
2. Never modify existing changesets (create new ones instead)
3. Use `logicalFilePath` attribute for refactoring safety

### Column Type Mapping
| Java Type | PostgreSQL Type | Use Case |
|-----------|-----------------|----------|
| `String` | `VARCHAR(255)` or `TEXT` | Text fields |
| `LocalDate` | `DATE` | Date-only fields (no time) |
| `OffsetDateTime` | `TIMESTAMP WITH TIME ZONE` | **Business date/time fields** (effectiveFrom, scheduledAt, etc.) |
| `Instant` | `TIMESTAMP WITH TIME ZONE` | **Audit timestamps** (createdAt, updatedAt) |
| `Integer` | `INTEGER` | Numbers |
| `Long` | `BIGINT` | Large numbers, IDs |
| `BigDecimal` | `DECIMAL(19,4)` | Money, precise numbers |
| `Boolean` | `BOOLEAN` | Flags |
| `Enum` | `VARCHAR(50)` | Enumerations |

### Date/Time Type Selection

**Use `OffsetDateTime` for business date/time fields:**
- Fields exposed through REST APIs (`effectiveFrom`, `effectiveTo`, `scheduledAt`, etc.)
- Business events or validity periods
- Any date field users interact with

**Rationale:**
- ✅ OpenAPI `format: date-time` generates `OffsetDateTime`
- ✅ **Zero mapping needed** between API DTOs and entities
- ✅ REST API standard (ISO-8601 with offset)

**Use `Instant` for audit timestamps:**
- JPA audit fields with `@CreatedDate`, `@LastModifiedDate`
- Internal system timestamps
- Technical, not business-relevant dates

**Example:**
```java
// Business dates - OffsetDateTime
@Column(name = "effective_from", nullable = false)
private OffsetDateTime effectiveFrom;

// Audit fields - Instant (JPA standard)
@CreatedDate
@Column(name = "created_at", nullable = false, updatable = false)
private Instant createdAt;
```

### Audit Fields: JPA vs Database Defaults

**CRITICAL RULE**: For audit fields (`createdAt`, `updatedAt`), use **JPA auditing as the single source of truth**, NOT database defaults.

**❌ DON'T: Use database defaults for audit fields**
```xml
<!-- BAD: Creates redundancy with JPA @CreatedDate -->
<column name="created_at" type="TIMESTAMP WITH TIME ZONE" defaultValueComputed="CURRENT_TIMESTAMP">
    <constraints nullable="false"/>
</column>
```

**✅ DO: Let JPA auditing handle it**
```xml
<!-- GOOD: JPA @CreatedDate manages the value -->
<column name="created_at" type="TIMESTAMP WITH TIME ZONE">
    <constraints nullable="false"/>
</column>
```

**Entity side:**
```java
@Column(name = "created_at", nullable = false, updatable = false)
@CreatedDate
private Instant createdAt;

@Column(name = "updated_at", nullable = false)
@LastModifiedDate
private Instant updatedAt;
```

**Why JPA auditing is preferred:**
- **Single source of truth**: Clear responsibility
- **Testability**: Can inject Clock for deterministic timestamps in tests
- **Portability**: Works across all databases
- **Consistency**: Same mechanism for `createdAt`, `updatedAt`, `createdBy`, `updatedBy`
- **Control**: Application has full control over audit values

**When to use database defaults:**
- Business fields with fixed defaults (e.g., `status` with default "PENDING")
- Simple default values like `active = true`
- Fields that truly should default at DB level (e.g., for direct SQL inserts)

## ArchUnit Compliance Checklist

Before marking implementation complete, verify:

- [ ] **Entity Naming**: Entity classes MUST have `Entity` suffix (e.g., `LimitEntity`, `OrderEntity`)
- [ ] **Entity in Domain**: Entity class lives in `domain/{feature}/`
- [ ] **Port in Domain**: Repository interface (port) lives in `domain/{feature}/`
- [ ] **Implementation in Infrastructure**: JPA repository lives in `infrastructure/persistence/`
- [ ] **Constructor Injection**: Repository implementations use `@RequiredArgsConstructor`
- [ ] **No Field Injection**: No `@Autowired` on fields (exception: `@PersistenceContext` for EntityManager)
- [ ] **No Deprecated APIs**: No deprecated JPA or Hibernate APIs
- [ ] **Java Time API**: Use `java.time.*` classes in entities
- [ ] **SLF4J Logging**: Use `@Slf4j` annotation
- [ ] **Test Class Naming**: Test classes MUST use `Test` suffix (e.g., `Jpa{Feature}RepositoryTest`), NOT `Spec`
- [ ] **TestContainers**: Repository tests must use TestContainers with `@Import(TestContainersConfiguration)`
- [ ] **ResourceLock**: Repository tests MUST use `@ResourceLock("database")` for parallel execution safety

## Database Patterns Reference

### Row-Level Locking
```sql
SELECT * FROM table WHERE condition FOR UPDATE SKIP LOCKED
```
- Use for concurrent processing of queue-like tables
- `SKIP LOCKED` prevents blocking on already-locked rows

### Claim Pattern
```sql
UPDATE table SET claimed_by = ?, claimed_at = NOW()
WHERE id = (SELECT id FROM table WHERE ... LIMIT 1)
RETURNING *
```
- Use for distributed job processing
- Include timeout for crash recovery

### Partitioning (PostgreSQL)
```sql
CREATE TABLE ... PARTITION BY RANGE (partition_date);
CREATE TABLE table_YYYYMMDD PARTITION OF table FOR VALUES FROM ('YYYY-MM-DD') TO ('YYYY-MM-DD');
```
- Use for time-series data with retention requirements
- Enables efficient partition pruning and maintenance

## Example Usage

When asked to create persistence for a new feature:

1. Create entity class in `domain/{feature}/`
2. Create composite ID if needed in `domain/{feature}/`
3. Create status enum in `domain/{feature}/`
4. Create repository interface (port) in `domain/{feature}/`
5. Create JPA repository in `infrastructure/persistence/`
6. Create custom batch repository if needed in `infrastructure/persistence/`
7. Create repository configuration in `infrastructure/persistence/`
8. Create Liquibase changelog
9. Add changelog to master
10. Create repository tests in `infrastructure/persistence/`

```
User: "Create database persistence for order processing"

Agent creates:
In domain/order/:
- OrderEntity.java (entity with Entity suffix, @EmbeddedId, Persistable)
- OrderEntityId.java (composite key)
- OrderStatus.java (enum)
- OrderRepository.java (port interface)

In infrastructure/persistence/:
- JpaOrderRepository.java (Spring Data, implements port)
- OrderBatchRepository.java (custom batch interface)
- OrderBatchRepositoryImpl.java (implementation)
- RepositoryConfiguration.java (beans)

In resources/db/changelog/:
- TICKET-create-orders-table.xml (Liquibase)
- db.changelog-master.xml (update)

In test/infrastructure/persistence/:
- JpaOrderRepositoryTest.groovy (Spock test)
```
