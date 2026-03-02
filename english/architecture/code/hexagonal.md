# 🔷 AI INSTRUCTION — Hexagonal Architecture (Ports & Adapters)
> Spring Boot · Java 21+ · Maven/Gradle

---

## 🎯 Architecture Context

You are an expert in Java software architecture with Spring Boot. Your task is to create a complete Spring Boot project strictly following the **Hexagonal Architecture (Ports & Adapters)**.

**Fundamental principle:** The application core (domain + use cases) is completely isolated from the outside world. All input and output access occurs via **Ports (interfaces)** and **Adapters (implementations)**.

---

## 📐 Package Structure

Generate the project with the following directory structure. Use `{base-package}` as a placeholder for the base package provided by the user:

```
src/
└── main/
    └── java/
        └── {base-package}/
            ├── core/                                  ← APPLICATION CORE
            │   ├── domain/                            ← Pure entities and rules
            │   │   ├── model/                         ← Domain models (entities, VOs)
            │   │   ├── exception/                     ← Domain exceptions
            │   │   └── event/                         ← Domain events (optional)
            │   ├── port/                              ← CONTRACTS (interfaces)
            │   │   ├── in/                            ← Primary ports (use cases)
            │   │   │   └── {UseCase}UseCase.java
            │   │   └── out/                           ← Secondary ports (repositories, external)
            │   │       └── {Entity}Repository.java
            │   └── usecase/                           ← Use case implementations
            │       └── {UseCase}UseCaseImpl.java
            │
            └── adapter/                               ← ADAPTERS
                ├── in/                                ← Primary adapters (entry points)
                │   ├── rest/                          ← REST Controllers
                │   │   ├── {Entity}Controller.java
                │   │   ├── request/
                │   │   │   └── {Entity}Request.java
                │   │   └── response/
                │   │       └── {Entity}Response.java
                │   ├── consumer/                      ← Kafka/RabbitMQ consumers (if applicable)
                │   └── scheduler/                     ← Scheduled tasks (if applicable)
                └── out/                               ← Secondary adapters (implementations)
                    ├── persistence/                   ← JPA/MongoDB adapters
                    │   ├── entity/
                    │   │   └── {Entity}JpaEntity.java
                    │   ├── mapper/
                    │   │   └── {Entity}PersistenceMapper.java
                    │   ├── repository/
                    │   │   └── {Entity}JpaRepository.java
                    │   └── {Entity}PersistenceAdapter.java
                    └── client/                        ← HTTP clients (Feign, WebClient)

src/
└── test/
    └── java/
        └── {base-package}/
            ├── core/
            │   └── usecase/                           ← Pure unit tests (no Spring)
            ├── adapter/
            │   ├── in/rest/                           ← @WebMvcTest per controller
            │   └── out/persistence/                   ← @DataJpaTest per adapter
            └── integration/                           ← @SpringBootTest
```

---

## 📋 Dependency Rules (MANDATORY)

```
adapter/in  →  core/port/in  →  core/usecase  →  core/port/out  ←  adapter/out
                                      ↓
                               core/domain/model
```

**Absolute rules — never violate:**
1. `core/` **NEVER** imports anything from `adapter/`
2. `core/domain/` **NEVER** imports Spring, JPA, or any framework
3. `core/usecase/` can only import `core/domain/` and `core/port/`
4. `adapter/in/` depends on `core/port/in/` (never on `usecase/` directly)
5. `adapter/out/` implements `core/port/out/`
6. Mappings always happen in **adapters**, never in the core

---

## 🗂️ Code Templates

### 1. Domain Model (core/domain/model)

```java
package {base-package}.core.domain.model;

import java.util.UUID;
import java.time.LocalDateTime;

/**
 * Domain Entity — no framework annotations.
 * Contains business rules and domain invariants.
 */
public class {Entity} {

    private final {Entity}Id id;
    private String name;
    private {Entity}Status status;
    private final LocalDateTime createdAt;

    // Private constructor — use factory methods
    private {Entity}(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.status = {Entity}Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
    }

    // Factory method for new entity
    public static {Entity} create(String name) {
        validateName(name);
        return new Builder()
            .id(new {Entity}Id(UUID.randomUUID()))
            .name(name)
            .build();
    }

    // Factory method for reconstitution (e.g., from database)
    public static {Entity} reconstitute({Entity}Id id, String name, {Entity}Status status, LocalDateTime createdAt) {
        // ...
    }

    // Business rules — expressive methods
    public void deactivate() {
        if (this.status == {Entity}Status.INACTIVE) {
            throw new {Entity}AlreadyInactiveException(this.id);
        }
        this.status = {Entity}Status.INACTIVE;
    }

    private static void validateName(String name) {
        if (name == null || name.isBlank()) {
            throw new InvalidEntityNameException("Name cannot be blank");
        }
    }

    // Getters (no setters — immutability preferred)
    public {Entity}Id getId() { return id; }
    public String getName() { return name; }
    public {Entity}Status getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    // Value Object for ID
    public record {Entity}Id(UUID value) {
        public {Entity}Id {
            if (value == null) throw new IllegalArgumentException("ID cannot be null");
        }
    }

    public static class Builder { /* ... */ }
}
```

### 2. Primary Port / Use Case Interface (core/port/in)

```java
package {base-package}.core.port.in;

/**
 * Primary Port — defines the contract of a use case.
 * Triggered by Primary Adapters (REST, Kafka consumer, etc.).
 */
public interface Create{Entity}UseCase {

    {Entity}Result execute(Create{Entity}Command command);

    /**
     * Command — use case input data (immutable).
     * Basic validations here; complex rules in the domain.
     */
    record Create{Entity}Command(String name, String description) {
        public Create{Entity}Command {
            if (name == null || name.isBlank()) {
                throw new IllegalArgumentException("Name is required");
            }
        }
    }

    /**
     * Result — use case output data.
     */
    record {Entity}Result(String id, String name, String status, String createdAt) {}
}
```

### 3. Secondary Port / Repository Interface (core/port/out)

```java
package {base-package}.core.port.out;

import {base-package}.core.domain.model.{Entity};
import {base-package}.core.domain.model.{Entity}.{Entity}Id;
import java.util.Optional;
import java.util.List;

/**
 * Secondary Port — contract the core requires for persistence.
 * Implemented by adapter/out/persistence.
 */
public interface {Entity}Repository {

    {Entity} save({Entity} entity);

    Optional<{Entity}> findById({Entity}Id id);

    List<{Entity}> findAll();

    boolean existsById({Entity}Id id);

    void delete({Entity}Id id);
}
```

### 4. Use Case Implementation (core/usecase)

```java
package {base-package}.core.usecase;

import {base-package}.core.domain.model.{Entity};
import {base-package}.core.port.in.Create{Entity}UseCase;
import {base-package}.core.port.out.{Entity}Repository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Use Case Implementation — orchestrates domain and ports.
 * @Service is allowed here as it's a dependency injection framework, not business.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class Create{Entity}UseCaseImpl implements Create{Entity}UseCase {

    private final {Entity}Repository repository;

    @Override
    @Transactional
    public {Entity}Result execute(Create{Entity}Command command) {
        log.info("Creating {entity} with name: {}", command.name());

        var entity = {Entity}.create(command.name());
        var saved = repository.save(entity);

        log.info("{Entity} created with id: {}", saved.getId().value());
        return toResult(saved);
    }

    private {Entity}Result toResult({Entity} entity) {
        return new {Entity}Result(
            entity.getId().value().toString(),
            entity.getName(),
            entity.getStatus().name(),
            entity.getCreatedAt().toString()
        );
    }
}
```

### 5. REST Controller — Primary Adapter (adapter/in/rest)

```java
package {base-package}.adapter.in.rest;

import {base-package}.adapter.in.rest.request.Create{Entity}Request;
import {base-package}.adapter.in.rest.response.{Entity}Response;
import {base-package}.core.port.in.Create{Entity}UseCase;
import {base-package}.core.port.in.Create{Entity}UseCase.Create{Entity}Command;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.net.URI;

/**
 * Primary Adapter — REST.
 * Converts HTTP ↔ Commands/Results. Contains no business logic.
 */
@RestController
@RequestMapping("/api/v1/{entities}")
@RequiredArgsConstructor
public class {Entity}Controller {

    private final Create{Entity}UseCase create{Entity}UseCase;

    @PostMapping
    public ResponseEntity<{Entity}Response> create(@Valid @RequestBody Create{Entity}Request request) {
        var command = new Create{Entity}Command(request.name(), request.description());
        var result = create{Entity}UseCase.execute(command);

        return ResponseEntity
            .created(URI.create("/api/v1/{entities}/" + result.id()))
            .body({Entity}Response.from(result));
    }
}
```

### 6. Request/Response DTOs (adapter/in/rest)

```java
// Request
package {base-package}.adapter.in.rest.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record Create{Entity}Request(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,

    @Size(max = 500)
    String description
) {}

// Response
package {base-package}.adapter.in.rest.response;

import {base-package}.core.port.in.Create{Entity}UseCase.{Entity}Result;

public record {Entity}Response(String id, String name, String status, String createdAt) {
    public static {Entity}Response from({Entity}Result result) {
        return new {Entity}Response(result.id(), result.name(), result.status(), result.createdAt());
    }
}
```

### 7. JPA Entity (adapter/out/persistence/entity)

```java
package {base-package}.adapter.out.persistence.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * JPA Entity — belongs to the adapter, never to the domain.
 */
@Entity
@Table(name = "{entities}")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class {Entity}JpaEntity {

    @Id
    @Column(name = "id", nullable = false, updatable = false)
    private UUID id;

    @Column(name = "name", nullable = false, length = 100)
    private String name;

    @Column(name = "description", length = 500)
    private String description;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

### 8. Persistence Mapper (adapter/out/persistence/mapper)

```java
package {base-package}.adapter.out.persistence.mapper;

import {base-package}.adapter.out.persistence.entity.{Entity}JpaEntity;
import {base-package}.core.domain.model.{Entity};
import {base-package}.core.domain.model.{Entity}.{Entity}Id;
import org.springframework.stereotype.Component;

/**
 * Mapper between Domain Model ↔ JPA Entity.
 * Never use MapStruct @Mapper directly on ports — always in this adapter.
 */
@Component
public class {Entity}PersistenceMapper {

    public {Entity}JpaEntity toJpaEntity({Entity} domain) {
        return {Entity}JpaEntity.builder()
            .id(domain.getId().value())
            .name(domain.getName())
            .status(domain.getStatus().name())
            .build();
    }

    public {Entity} toDomain({Entity}JpaEntity jpaEntity) {
        return {Entity}.reconstitute(
            new {Entity}Id(jpaEntity.getId()),
            jpaEntity.getName(),
            {Entity}Status.valueOf(jpaEntity.getStatus()),
            jpaEntity.getCreatedAt()
        );
    }
}
```

### 9. Persistence Adapter — Secondary Adapter (adapter/out/persistence)

```java
package {base-package}.adapter.out.persistence;

import {base-package}.adapter.out.persistence.mapper.{Entity}PersistenceMapper;
import {base-package}.adapter.out.persistence.repository.{Entity}JpaRepository;
import {base-package}.core.domain.model.{Entity};
import {base-package}.core.domain.model.{Entity}.{Entity}Id;
import {base-package}.core.port.out.{Entity}Repository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.Optional;

/**
 * Secondary Adapter — implements the repository port.
 * Translates between JPA and Domain Model.
 */
@Component
@RequiredArgsConstructor
public class {Entity}PersistenceAdapter implements {Entity}Repository {

    private final {Entity}JpaRepository jpaRepository;
    private final {Entity}PersistenceMapper mapper;

    @Override
    public {Entity} save({Entity} entity) {
        var jpaEntity = mapper.toJpaEntity(entity);
        var saved = jpaRepository.save(jpaEntity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<{Entity}> findById({Entity}Id id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public List<{Entity}> findAll() {
        return jpaRepository.findAll().stream()
            .map(mapper::toDomain)
            .toList();
    }

    @Override
    public boolean existsById({Entity}Id id) {
        return jpaRepository.existsById(id.value());
    }

    @Override
    public void delete({Entity}Id id) {
        jpaRepository.deleteById(id.value());
    }
}
```

### 10. Spring Data Repository (adapter/out/persistence/repository)

```java
package {base-package}.adapter.out.persistence.repository;

import {base-package}.adapter.out.persistence.entity.{Entity}JpaEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.UUID;

/**
 * Spring Data JPA — infrastructure detail.
 * Stays completely inside adapter/out.
 */
public interface {Entity}JpaRepository extends JpaRepository<{Entity}JpaEntity, UUID> {
    // Custom queries here if needed
}
```

---

## 🧪 Testing Strategy

### Use Case Test (no Spring — JUnit 5 + Mockito)

```java
@ExtendWith(MockitoExtension.class)
class Create{Entity}UseCaseImplTest {

    @Mock
    private {Entity}Repository repository;

    @InjectMocks
    private Create{Entity}UseCaseImpl useCase;

    @Test
    void shouldCreate{Entity}Successfully() {
        // Arrange
        var command = new Create{Entity}Command("Test Name", "Description");
        when(repository.save(any({Entity}.class))).thenAnswer(inv -> inv.getArgument(0));

        // Act
        var result = useCase.execute(command);

        // Assert
        assertThat(result.name()).isEqualTo("Test Name");
        assertThat(result.status()).isEqualTo("ACTIVE");
        verify(repository).save(any({Entity}.class));
    }
}
```

### Controller Test (Spring MVC slice)

```java
@WebMvcTest({Entity}Controller.class)
class {Entity}ControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean Create{Entity}UseCase create{Entity}UseCase;

    @Test
    void shouldReturn201WhenCreating{Entity}() throws Exception {
        var result = new {Entity}Result(UUID.randomUUID().toString(), "Test", "ACTIVE", LocalDateTime.now().toString());
        when(create{Entity}UseCase.execute(any())).thenReturn(result);

        mockMvc.perform(post("/api/v1/{entities}")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name": "Test", "description": "Desc"}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("Test"));
    }
}
```

### Persistence Adapter Test (JPA slice)

```java
@DataJpaTest
@Import({Entity}PersistenceAdapter.class, {Entity}PersistenceMapper.class)
class {Entity}PersistenceAdapterTest {

    @Autowired
    private {Entity}PersistenceAdapter adapter;

    @Test
    void shouldSaveAndRetrieve{Entity}() {
        var entity = {Entity}.create("Test");
        var saved = adapter.save(entity);
        var found = adapter.findById(saved.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Test");
    }
}
```

---

## 📦 pom.xml — Essential Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
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

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## ✅ Project Generation Checklist

Before finalizing the project, **verify each item:**

- [ ] Domain model with no framework annotations (`@Entity`, `@Component`, etc.)
- [ ] Ports (`port/in` and `port/out`) are pure interfaces with no framework dependencies
- [ ] Use Cases do not know about `@RestController`, `JpaRepository`, or anything from `adapter/`
- [ ] Adapters handle mapping — core never receives DTOs or JPA entities directly
- [ ] `@Service` and `@Component` exist only in `usecase/` and `adapter/`
- [ ] Domain exceptions created in `core/domain/exception/`
- [ ] `GlobalExceptionHandler` created in `adapter/in/rest/` (not in the core)
- [ ] Use case tests run **without Spring Context** (JUnit + Mockito only)
- [ ] Controller tests use `@WebMvcTest` (not `@SpringBootTest`)
- [ ] Persistence tests use `@DataJpaTest` (not `@SpringBootTest`)
- [ ] Flyway migrations created in `resources/db/migration/`
- [ ] `application.yml` configured with profiles (`dev`, `test`, `prod`)

---

## 🚨 Common Mistakes — Never Do This

```java
// ❌ WRONG: domain model with JPA
@Entity  // Never in core/domain/model
public class {Entity} { ... }

// ❌ WRONG: use case knows the controller
@RestController  // Never in core/usecase
public class Create{Entity}UseCase { ... }

// ❌ WRONG: controller calls repository directly
@RestController
public class {Entity}Controller {
    @Autowired {Entity}JpaRepository jpaRepository; // total bypass of the architecture
}

// ❌ WRONG: output port with JPA dependency
public interface {Entity}Repository extends JpaRepository<...> {} // port/out cannot have this

// ✅ CORRECT: port is a pure interface
public interface {Entity}Repository {
    {Entity} save({Entity} entity);
}
```
