# 🔵 AI INSTRUCTION — Onion Architecture
> Spring Boot · Java 21+ · Maven/Gradle · DDD · Jeffrey Palermo

---

## 🎯 Architecture Context

You are an expert in Java software architecture with Spring Boot and Domain-Driven Design. Your task is to create a complete Spring Boot project strictly following the **Onion Architecture** by Jeffrey Palermo, strongly aligned with **DDD (Domain-Driven Design)** principles.

**Fundamental principle:** The **Domain Model** is the central core and depends on nothing. Layers grow around it like the layers of an onion, always pointing inward. Infrastructure lives in the outermost layer and is completely replaceable.

**Layers (from inside to outside):**
1. **Domain Model** — Entities, Value Objects, Aggregates, Domain Events, Enums
2. **Domain Services** — Services that operate on the Domain Model (stateless, no infra)
3. **Application Services** — Orchestrates Domain Services and Repositories (use cases)
4. **Infrastructure / UI** — JPA, REST, Kafka, HTTP clients, everything external

**Key difference from Clean:** Onion explicitly uses DDD terminology and concepts (Aggregates, Value Objects, Repositories as domain interfaces, Domain Events).

---

## 📐 Package Structure

```
src/
└── main/
    └── java/
        └── {base-package}/
            │
            ├── domain/                                ← LAYER 1: DOMAIN MODEL
            │   └── {boundedcontext}/
            │       ├── model/
            │       │   ├── {Aggregate}.java           ← Aggregate Root
            │       │   ├── {Entity}.java              ← Entity (within the aggregate)
            │       │   └── valueobject/
            │       │       ├── {Entity}Id.java        ← Value Object - ID
            │       │       ├── Money.java             ← Value Object - example
            │       │       └── Email.java             ← Value Object - example
            │       ├── event/
            │       │   └── {Entity}CreatedEvent.java  ← Domain Event
            │       ├── repository/
            │       │   └── {Aggregate}Repository.java ← Repository Interface (domain)
            │       ├── service/
            │       │   └── {Entity}DomainService.java ← Domain Service
            │       └── exception/
            │           ├── {Entity}NotFoundException.java
            │           └── {Entity}BusinessException.java
            │
            ├── application/                           ← LAYER 2: APPLICATION SERVICES
            │   └── {boundedcontext}/
            │       ├── service/
            │       │   └── {UseCase}ApplicationService.java
            │       └── dto/
            │           ├── command/
            │           │   └── Create{Entity}Command.java
            │           └── query/
            │               └── {Entity}QueryResult.java
            │
            └── infrastructure/                        ← LAYER 3: INFRASTRUCTURE
                ├── persistence/
                │   ├── entity/
                │   │   └── {Entity}OrmEntity.java     ← JPA Entity
                │   ├── repository/
                │   │   ├── {Aggregate}JpaRepository.java ← Spring Data JPA
                │   │   └── {Aggregate}RepositoryImpl.java ← implements domain repository
                │   └── mapper/
                │       └── {Entity}OrmMapper.java
                ├── web/
                │   ├── controller/
                │   │   ├── {Entity}Controller.java
                │   │   ├── request/
                │   │   └── response/
                │   └── exception/
                │       └── GlobalExceptionHandler.java
                ├── messaging/
                │   ├── publisher/
                │   │   └── {Entity}EventPublisher.java
                │   └── consumer/
                │       └── {Entity}EventConsumer.java
                └── config/
                    ├── ApplicationConfig.java
                    └── InfrastructureConfig.java

src/
└── test/
    └── java/
        └── {base-package}/
            ├── domain/                                ← Pure JUnit — isolated domain
            ├── application/                           ← JUnit + Mockito
            └── infrastructure/
                ├── persistence/                       ← @DataJpaTest
                ├── web/                               ← @WebMvcTest
                └── integration/                       ← @SpringBootTest + Testcontainers
```

---

## 📋 Dependency Rules (MANDATORY)

```
[Domain Model]         ← zero external dependencies
      ↑
[Domain Services]      ← depends only on Domain Model
      ↑
[Application Services] ← depends on Domain + Domain Services
      ↑
[Infrastructure]       ← depends on everything, implements domain interfaces
```

**Absolute DDD rules — never violate:**
1. `domain/model/` has **zero external dependencies** — pure Java
2. `domain/repository/` are interfaces — never depend on JPA or Spring
3. `domain/service/` does not access databases directly
4. `application/service/` does not access JPA or infrastructure directly
5. `infrastructure/` implements contracts defined in `domain/`
6. Aggregates encapsulate invariants — never expose mutable internal collections

---

## 🗂️ DDD Building Blocks & Templates

### 1. Value Object (domain/{boundedcontext}/model/valueobject)

```java
package {base-package}.domain.{boundedcontext}.model.valueobject;

import java.util.Objects;
import java.util.UUID;

/**
 * Value Object — immutable, no identity, equality by value.
 * Use records for simple VOs (Java 16+).
 */
public record {Entity}Id(UUID value) {

    public {Entity}Id {
        Objects.requireNonNull(value, "{Entity}Id cannot be null");
    }

    public static {Entity}Id generate() {
        return new {Entity}Id(UUID.randomUUID());
    }

    public static {Entity}Id of(String value) {
        return new {Entity}Id(UUID.fromString(value));
    }

    @Override
    public String toString() {
        return value.toString();
    }
}

// Example Value Object with business rule
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Money amount cannot be negative");
        amount = amount.setScale(2, RoundingMode.HALF_UP);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Cannot add different currencies");
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        return this.amount.compareTo(other.amount) > 0;
    }
}
```

### 2. Aggregate Root (domain/{boundedcontext}/model)

```java
package {base-package}.domain.{boundedcontext}.model;

import {base-package}.domain.{boundedcontext}.event.{Entity}CreatedEvent;
import {base-package}.domain.{boundedcontext}.exception.{Entity}BusinessException;
import {base-package}.domain.{boundedcontext}.model.valueobject.{Entity}Id;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Aggregate Root — entry point for the aggregate.
 * Protects invariants and emits Domain Events.
 * Child entities are only accessed via the Aggregate Root.
 */
public class {Entity} {

    private final {Entity}Id id;
    private String name;
    private {Entity}Status status;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Domain Events — accumulated for publishing after persistence
    private final List<Object> domainEvents = new ArrayList<>();

    // Private constructor — use factory methods
    private {Entity}({Entity}Id id, String name) {
        this.id = id;
        this.name = name;
        this.status = {Entity}Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    /**
     * Factory Method — creation with invariant validation.
     */
    public static {Entity} create(String name) {
        validateName(name);
        var entity = new {Entity}({Entity}Id.generate(), name);
        entity.registerEvent(new {Entity}CreatedEvent(entity.id, entity.name, entity.createdAt));
        return entity;
    }

    /**
     * Reconstitution — used by repositories to restore state.
     */
    public static {Entity} reconstitute({Entity}Id id, String name, {Entity}Status status,
                                        LocalDateTime createdAt, LocalDateTime updatedAt) {
        var entity = new {Entity}(id, name);
        entity.status = status;
        entity.updatedAt = updatedAt;
        return entity;
    }

    // Domain behaviors (expressive methods — Ubiquitous Language)
    public void activate() {
        if (isActive()) throw new {Entity}BusinessException("Already active", this.id);
        this.status = {Entity}Status.ACTIVE;
        this.updatedAt = LocalDateTime.now();
    }

    public void suspend() {
        if (!isActive()) throw new {Entity}BusinessException("Cannot suspend non-active entity", this.id);
        this.status = {Entity}Status.SUSPENDED;
        this.updatedAt = LocalDateTime.now();
    }

    public boolean isActive() {
        return this.status == {Entity}Status.ACTIVE;
    }

    public void rename(String newName) {
        validateName(newName);
        this.name = newName;
        this.updatedAt = LocalDateTime.now();
    }

    private static void validateName(String name) {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name cannot be blank");
        if (name.length() > 100)
            throw new {Entity}BusinessException("Name exceeds 100 characters", null);
    }

    private void registerEvent(Object event) {
        this.domainEvents.add(event);
    }

    // Getters
    public {Entity}Id getId() { return id; }
    public String getName() { return name; }
    public {Entity}Status getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }

    // Returns immutable copy — does not expose mutable internal list
    public List<Object> pullDomainEvents() {
        var events = Collections.unmodifiableList(new ArrayList<>(domainEvents));
        domainEvents.clear();
        return events;
    }
}
```

### 3. Domain Event (domain/{boundedcontext}/event)

```java
package {base-package}.domain.{boundedcontext}.event;

import {base-package}.domain.{boundedcontext}.model.valueobject.{Entity}Id;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Domain Event — represents something that happened in the domain.
 * Immutable, named in the past tense.
 */
public record {Entity}CreatedEvent(
    UUID eventId,
    {Entity}Id {entity}Id,
    String {entity}Name,
    LocalDateTime occurredAt
) {
    public {Entity}CreatedEvent({Entity}Id {entity}Id, String {entity}Name, LocalDateTime occurredAt) {
        this(UUID.randomUUID(), {entity}Id, {entity}Name, occurredAt);
    }
}
```

### 4. Repository Interface (domain/{boundedcontext}/repository)

```java
package {base-package}.domain.{boundedcontext}.repository;

import {base-package}.domain.{boundedcontext}.model.{Entity};
import {base-package}.domain.{boundedcontext}.model.valueobject.{Entity}Id;
import java.util.List;
import java.util.Optional;

/**
 * Repository — interface defined in the domain.
 * Domain language (not generic "save/find", but expressive methods).
 * Implemented by infrastructure.
 */
public interface {Entity}Repository {

    void store({Entity} {entity});          // Prefer domain names

    Optional<{Entity}> findById({Entity}Id id);

    List<{Entity}> findAllActive();

    boolean exists({Entity}Id id);

    void remove({Entity}Id id);
}
```

### 5. Domain Service (domain/{boundedcontext}/service)

```java
package {base-package}.domain.{boundedcontext}.service;

import {base-package}.domain.{boundedcontext}.model.{Entity};
import {base-package}.domain.{boundedcontext}.repository.{Entity}Repository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

/**
 * Domain Service — domain logic that does not belong to a single entity.
 * Stateless. Can use Repository interfaces from the domain.
 * Does NOT orchestrate use cases — that is the Application Service's role.
 */
@Service
@RequiredArgsConstructor
public class {Entity}DomainService {

    private final {Entity}Repository {entity}Repository;

    /**
     * Checks if the name already exists in the domain (uniqueness rule).
     */
    public boolean isNameUnique(String name) {
        // Implementation with repository query
        return true; // simplified example
    }

    /**
     * Example: complex logic involving multiple entities.
     */
    public void validateBusinessRule({Entity} {entity}) {
        // Business rule that needs context beyond the entity itself
    }
}
```

### 6. Application Service (application/{boundedcontext}/service)

```java
package {base-package}.application.{boundedcontext}.service;

import {base-package}.application.{boundedcontext}.dto.command.Create{Entity}Command;
import {base-package}.application.{boundedcontext}.dto.query.{Entity}QueryResult;
import {base-package}.domain.{boundedcontext}.model.{Entity};
import {base-package}.domain.{boundedcontext}.repository.{Entity}Repository;
import {base-package}.domain.{boundedcontext}.service.{Entity}DomainService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Application Service — orchestrates the flow of a use case.
 * Coordinates Domain Services and Repositories.
 * Publishes Domain Events after transaction.
 * Does NOT contain business rules — delegates to the domain.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class {Entity}ApplicationService {

    private final {Entity}Repository {entity}Repository;
    private final {Entity}DomainService {entity}DomainService;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public {Entity}QueryResult create{Entity}(Create{Entity}Command command) {
        log.info("Creating {entity}: {}", command.name());

        // Business rule validation via Domain Service
        if (!{entity}DomainService.isNameUnique(command.name())) {
            throw new IllegalArgumentException("{Entity} name already exists: " + command.name());
        }

        // Creation via Aggregate factory method
        var {entity} = {Entity}.create(command.name());

        // Persistence via domain Repository
        {entity}Repository.store({entity});

        // Publication of Domain Events
        {entity}.pullDomainEvents().forEach(event -> {
            log.debug("Publishing domain event: {}", event.getClass().getSimpleName());
            eventPublisher.publishEvent(event);
        });

        log.info("{Entity} created: {}", {entity}.getId());
        return toQueryResult({entity});
    }

    @Transactional(readOnly = true)
    public {Entity}QueryResult find{Entity}ById(String id) {
        return {entity}Repository.findById(
                {Entity}Id.of(id))
            .map(this::toQueryResult)
            .orElseThrow(() -> new {Entity}NotFoundException(id));
    }

    private {Entity}QueryResult toQueryResult({Entity} {entity}) {
        return new {Entity}QueryResult(
            {entity}.getId().toString(),
            {entity}.getName(),
            {entity}.getStatus().name(),
            {entity}.getCreatedAt(),
            {entity}.getUpdatedAt()
        );
    }
}
```

### 7. Command & Query DTOs (application/{boundedcontext}/dto)

```java
// Command — intent to change state
package {base-package}.application.{boundedcontext}.dto.command;

public record Create{Entity}Command(String name, String description) {
    public Create{Entity}Command {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name is required");
    }
}

// Query Result — read result
package {base-package}.application.{boundedcontext}.dto.query;

import java.time.LocalDateTime;

public record {Entity}QueryResult(
    String id,
    String name,
    String status,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

### 8. ORM Entity (infrastructure/persistence/entity)

```java
package {base-package}.infrastructure.persistence.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * ORM Entity — persistence detail, not the Domain Model.
 * Uses a different name ({Entity}OrmEntity) to avoid confusion.
 */
@Entity
@Table(name = "{entities}")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class {Entity}OrmEntity {

    @Id
    private UUID id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(length = 500)
    private String description;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private String status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @Version
    private Long version;  // Optimistic locking — DDD recommended

    @PrePersist
    void onPersist() {
        createdAt = LocalDateTime.now();
        updatedAt = createdAt;
    }

    @PreUpdate
    void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### 9. Repository Implementation (infrastructure/persistence/repository)

```java
package {base-package}.infrastructure.persistence.repository;

import {base-package}.domain.{boundedcontext}.model.{Entity};
import {base-package}.domain.{boundedcontext}.model.valueobject.{Entity}Id;
import {base-package}.domain.{boundedcontext}.repository.{Entity}Repository;
import {base-package}.infrastructure.persistence.mapper.{Entity}OrmMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

/**
 * Repository Implementation — in infrastructure, implements domain interface.
 * Translates between Domain Aggregate ↔ ORM Entity.
 */
@Repository
@RequiredArgsConstructor
public class {Entity}RepositoryImpl implements {Entity}Repository {

    private final {Entity}JpaRepository jpaRepository;
    private final {Entity}OrmMapper mapper;

    @Override
    public void store({Entity} {entity}) {
        var ormEntity = mapper.toOrm({entity});
        jpaRepository.save(ormEntity);
    }

    @Override
    public Optional<{Entity}> findById({Entity}Id id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public List<{Entity}> findAllActive() {
        return jpaRepository.findByStatus("ACTIVE")
            .stream()
            .map(mapper::toDomain)
            .toList();
    }

    @Override
    public boolean exists({Entity}Id id) {
        return jpaRepository.existsById(id.value());
    }

    @Override
    public void remove({Entity}Id id) {
        jpaRepository.deleteById(id.value());
    }
}
```

### 10. Spring Data JPA Repository (infrastructure/persistence/repository)

```java
package {base-package}.infrastructure.persistence.repository;

import {base-package}.infrastructure.persistence.entity.{Entity}OrmEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.UUID;

@Repository
public interface {Entity}JpaRepository extends JpaRepository<{Entity}OrmEntity, UUID> {
    List<{Entity}OrmEntity> findByStatus(String status);
}
```

### 11. REST Controller (infrastructure/web/controller)

```java
package {base-package}.infrastructure.web.controller;

import {base-package}.application.{boundedcontext}.dto.command.Create{Entity}Command;
import {base-package}.application.{boundedcontext}.dto.query.{Entity}QueryResult;
import {base-package}.application.{boundedcontext}.service.{Entity}ApplicationService;
import {base-package}.infrastructure.web.request.Create{Entity}Request;
import {base-package}.infrastructure.web.response.{Entity}Response;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.net.URI;

/**
 * REST Controller — UI infrastructure.
 * Converts HTTP ↔ Application Commands/Queries.
 */
@RestController
@RequestMapping("/api/v1/{entities}")
@RequiredArgsConstructor
public class {Entity}Controller {

    private final {Entity}ApplicationService applicationService;

    @PostMapping
    public ResponseEntity<{Entity}Response> create(@Valid @RequestBody Create{Entity}Request request) {
        var command = new Create{Entity}Command(request.name(), request.description());
        var result = applicationService.create{Entity}(command);

        return ResponseEntity
            .created(URI.create("/api/v1/{entities}/" + result.id()))
            .body({Entity}Response.from(result));
    }

    @GetMapping("/{id}")
    public ResponseEntity<{Entity}Response> findById(@PathVariable String id) {
        var result = applicationService.find{Entity}ById(id);
        return ResponseEntity.ok({Entity}Response.from(result));
    }
}
```

### 12. Domain Event Handler (infrastructure/messaging)

```java
package {base-package}.infrastructure.messaging.publisher;

import {base-package}.domain.{boundedcontext}.event.{Entity}CreatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

/**
 * Domain Event Handler — processes Domain Events after transaction commit.
 * Can publish to Kafka, send emails, notify other bounded contexts.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class {Entity}EventPublisher {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on{Entity}Created({Entity}CreatedEvent event) {
        log.info("Handling domain event: {Entity}Created for id={}", event.{entity}Id());
        // Publish to Kafka, webhook, etc.
    }
}
```

---

## 🧪 DDD Testing Strategy

### Domain Model Test (pure JUnit — zero mocks)

```java
class {Entity}Test {

    @Test
    void shouldCreateAggregateWithActiveStatus() {
        var {entity} = {Entity}.create("Test Name");
        assertThat({entity}.isActive()).isTrue();
        assertThat({entity}.getId()).isNotNull();
        assertThat({entity}.pullDomainEvents()).hasSize(1)
            .first().isInstanceOf({Entity}CreatedEvent.class);
    }

    @Test
    void shouldRegisterDomainEventOnCreation() {
        var {entity} = {Entity}.create("Test");
        var events = {entity}.pullDomainEvents();
        assertThat(events).hasSize(1);

        var event = ({Entity}CreatedEvent) events.get(0);
        assertThat(event.{entity}Name()).isEqualTo("Test");
    }

    @Test
    void shouldClearEventsAfterPull() {
        var {entity} = {Entity}.create("Test");
        {entity}.pullDomainEvents(); // first pull
        assertThat({entity}.pullDomainEvents()).isEmpty(); // second pull
    }

    @Test
    void shouldThrowWhenCreatingWithBlankName() {
        assertThatThrownBy(() -> {Entity}.create(""))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

### Application Service Test (Mockito)

```java
@ExtendWith(MockitoExtension.class)
class {Entity}ApplicationServiceTest {

    @Mock {Entity}Repository {entity}Repository;
    @Mock {Entity}DomainService {entity}DomainService;
    @Mock ApplicationEventPublisher eventPublisher;

    @InjectMocks {Entity}ApplicationService service;

    @Test
    void shouldCreate{Entity}WhenNameIsUnique() {
        // Arrange
        when({entity}DomainService.isNameUnique(any())).thenReturn(true);
        doNothing().when({entity}Repository).store(any());

        // Act
        var result = service.create{Entity}(new Create{Entity}Command("Test", "Desc"));

        // Assert
        assertThat(result.name()).isEqualTo("Test");
        assertThat(result.status()).isEqualTo("ACTIVE");
        verify({entity}Repository).store(any({Entity}.class));
        verify(eventPublisher).publishEvent(any({Entity}CreatedEvent.class));
    }

    @Test
    void shouldThrowWhenNameIsNotUnique() {
        when({entity}DomainService.isNameUnique(any())).thenReturn(false);
        assertThatThrownBy(() -> service.create{Entity}(new Create{Entity}Command("Duplicate", null)))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

---

## 📦 Recommended DDD Dependencies (pom.xml)

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>

    <!-- Utilities -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.5.5.Final</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## ✅ DDD + Onion — Project Generation Checklist

- [ ] Domain Model with no framework annotations or external imports
- [ ] Value Objects are immutable (records or final fields + no setters)
- [ ] Aggregate Root is the only entry point for the Aggregate (private setter)
- [ ] Domain Events created as immutable records named in the past tense
- [ ] `pullDomainEvents()` clears events after returning them
- [ ] Repository interfaces in `domain/` use ubiquitous language
- [ ] Domain Services do not access infrastructure directly
- [ ] Application Services orchestrate — do not contain domain rules
- [ ] `@TransactionalEventListener(phase = AFTER_COMMIT)` for Domain Events
- [ ] ORM Entities named differently from Domain Entities (e.g., `OrmEntity`)
- [ ] Mappers between Domain ↔ ORM isolated in `infrastructure/persistence/mapper/`
- [ ] Domain tests running without Spring Context
- [ ] Application Service tests with Mockito (without Spring)
- [ ] Persistence tests with `@DataJpaTest` + H2/Testcontainers
- [ ] Bounded Contexts well defined in `domain/` (subpackages per context)

---

## 🚨 Common Mistakes — Never Do This

```java
// ❌ WRONG: Aggregate with JPA annotation
@Entity  // NEVER in domain/model
public class {Entity} {}

// ❌ WRONG: Domain Service accesses infrastructure
@Service
public class {Entity}DomainService {
    @Autowired EntityManager em;  // violates domain isolation
}

// ❌ WRONG: Application Service contains business rule
@Service
public class {Entity}ApplicationService {
    public void create{Entity}(Create{Entity}Command cmd) {
        if (cmd.name().length() > 100) throw ...;  // this is a domain rule!
    }
}

// ❌ WRONG: Controller accesses Domain Service directly
@RestController
public class {Entity}Controller {
    @Autowired {Entity}DomainService domainService;  // controller only accesses Application Service
}

// ❌ WRONG: Domain repository extends JpaRepository
public interface {Entity}Repository extends JpaRepository<...> {}  // domain cannot depend on JPA

// ✅ CORRECT: Domain repository is a pure interface
public interface {Entity}Repository {
    void store({Entity} {entity});
    Optional<{Entity}> findById({Entity}Id id);
}

// ✅ CORRECT: Domain Event published after commit
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void on{Entity}Created({Entity}CreatedEvent event) { ... }
```

---

## 🗣️ DDD Glossary (Ubiquitous Language)

| Term | Meaning | Implementation |
|---|---|---|
| **Aggregate** | Cluster of entities treated as a unit | Class with factory method |
| **Aggregate Root** | Entry point of the Aggregate | Main entity with `create()` |
| **Value Object** | Object without identity, equal by value | Java `record` |
| **Domain Event** | Something that happened in the domain | `record` named in past tense |
| **Repository** | Abstraction of an Aggregate collection | Interface in the domain |
| **Domain Service** | Logic that doesn't belong to a single entity | `@Service` in domain |
| **Application Service** | Orchestrates use cases | `@Service` in application/ |
| **Bounded Context** | Boundary of a subdomain | Subpackage in domain/ |
| **Ubiquitous Language** | Shared language with domain experts | Method/class names |
