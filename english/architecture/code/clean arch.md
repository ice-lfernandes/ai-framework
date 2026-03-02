# 🔶 AI INSTRUCTION — Clean Architecture
> Spring Boot · Java 21+ · Maven/Gradle · Robert C. Martin

---

## 🎯 Architecture Context

You are an expert in Java software architecture with Spring Boot. Your task is to create a complete Spring Boot project strictly following the **Clean Architecture** by Robert C. Martin (Uncle Bob).

**Fundamental principle:** Dependencies always point inward. The innermost layers (Entities and Use Cases) know nothing about the outer layers. The **Dependency Rule** is absolute and inviolable.

**Layers (from inside to outside):**
1. **Entities** — corporate business rules (independent of any application)
2. **Use Cases** — application business rules (orchestrates Entities)
3. **Interface Adapters** — Controllers, Presenters, Gateways (converts formats)
4. **Frameworks & Drivers** — Web, DB, UI, external devices

---

## 📐 Package Structure

```
src/
└── main/
    └── java/
        └── {base-package}/
            │
            ├── enterprise/                            ← LAYER 1: ENTITIES
            │   └── {domain}/
            │       ├── {Entity}.java                  ← Business Entity
            │       ├── {Entity}Id.java                ← Value Object
            │       ├── {Entity}Status.java            ← Domain Enum
            │       └── exception/
            │           └── {Entity}NotFoundException.java
            │
            ├── application/                           ← LAYER 2: USE CASES
            │   └── {domain}/
            │       ├── port/
            │       │   ├── in/                        ← Input Boundaries
            │       │   │   ├── {UseCase}InputPort.java
            │       │   │   └── dto/
            │       │   │       ├── {UseCase}InputData.java
            │       │   │       └── {UseCase}OutputData.java
            │       │   └── out/                       ← Output Boundaries / Gateways
            │       │       └── {Entity}Gateway.java
            │       └── interactor/
            │           └── {UseCase}Interactor.java   ← Use Case implementation
            │
            ├── adapter/                               ← LAYER 3: INTERFACE ADAPTERS
            │   ├── controller/                        ← REST Controllers
            │   │   ├── {Entity}Controller.java
            │   │   ├── request/
            │   │   └── response/
            │   ├── presenter/                         ← Response formatters (optional)
            │   │   └── {Entity}Presenter.java
            │   └── gateway/                           ← Gateway implementations (DB, API)
            │       ├── {Entity}GatewayImpl.java
            │       ├── mapper/
            │       │   └── {Entity}DataMapper.java
            │       └── dataaccess/
            │           ├── {Entity}DataAccessObject.java
            │           └── {Entity}JpaRepository.java
            │
            └── infrastructure/                        ← LAYER 4: FRAMEWORKS & DRIVERS
                ├── config/
            │   ├── ApplicationConfig.java
            │   ├── SecurityConfig.java
            │   └── OpenApiConfig.java
                ├── persistence/
            │   └── entity/
            │       └── {Entity}DataModel.java         ← JPA Entity
                └── web/
                    └── exception/
                        └── GlobalExceptionHandler.java

src/
└── test/
    └── java/
        └── {base-package}/
            ├── enterprise/                            ← Entity Tests (pure Java)
            ├── application/
            │   └── {domain}/interactor/               ← Use Case Tests (Mockito)
            ├── adapter/
            │   ├── controller/                        ← @WebMvcTest
            │   └── gateway/                           ← @DataJpaTest
            └── integration/                           ← @SpringBootTest (E2E)
```

---

## 📋 Dependency Rule (MANDATORY)

```
[Enterprise Entities]
        ↑
[Application Use Cases]   ←  defines Gateways as interfaces
        ↑
[Interface Adapters]      ←  implements Gateways, uses Use Cases
        ↑
[Frameworks & Drivers]    ←  Spring, JPA, HTTP, Kafka
```

**Absolute rules — never violate:**
1. `enterprise/` **NEVER** imports anything from `application/`, `adapter/` or `infrastructure/`
2. `application/` **NEVER** imports anything from `adapter/` or `infrastructure/`
3. `application/` defines **Gateways** (interfaces) — `adapter/` implements them
4. `adapter/` cannot directly import `infrastructure/` (only via interfaces)
5. Data crosses boundaries as simple **Data Transfer Objects** (records/structs)
6. Data flow follows: Controller → InputPort → Interactor → Gateway → DataModel

---

## 🗂️ Code Templates

### 1. Enterprise Entity (enterprise/{domain})

```java
package {base-package}.enterprise.{domain};

import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Enterprise Business Entity.
 * Contains the most critical and stable business rules.
 * Zero external dependencies — pure Java.
 */
public class {Entity} {

    private final {Entity}Id id;
    private String name;
    private {Entity}Status status;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    private {Entity}(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.status = {Entity}Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    public static {Entity} create(String name) {
        validate(name);
        return new Builder()
            .id(new {Entity}Id(UUID.randomUUID()))
            .name(name)
            .build();
    }

    public static {Entity} restore({Entity}Id id, String name, {Entity}Status status,
                                   LocalDateTime createdAt, LocalDateTime updatedAt) {
        var entity = new {Entity}(new Builder().id(id).name(name));
        entity.status = status;
        entity.updatedAt = updatedAt;
        return entity;
    }

    // Enterprise business rules
    public void activate() {
        if (this.status == {Entity}Status.ACTIVE)
            throw new IllegalStateException("{Entity} is already active");
        this.status = {Entity}Status.ACTIVE;
        this.updatedAt = LocalDateTime.now();
    }

    public void deactivate() {
        if (this.status == {Entity}Status.INACTIVE)
            throw new IllegalStateException("{Entity} is already inactive");
        this.status = {Entity}Status.INACTIVE;
        this.updatedAt = LocalDateTime.now();
    }

    private static void validate(String name) {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name cannot be blank");
        if (name.length() > 100)
            throw new IllegalArgumentException("Name exceeds maximum length");
    }

    public {Entity}Id getId() { return id; }
    public String getName() { return name; }
    public {Entity}Status getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }

    public record {Entity}Id(UUID value) {
        public {Entity}Id { if (value == null) throw new IllegalArgumentException(); }
        public static {Entity}Id generate() { return new {Entity}Id(UUID.randomUUID()); }
        @Override public String toString() { return value.toString(); }
    }

    public static class Builder {
        private {Entity}Id id;
        private String name;
        public Builder id({Entity}Id id) { this.id = id; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public {Entity} build() { return new {Entity}(this); }
    }
}
```

### 2. Input Boundary / Use Case Interface (application/{domain}/port/in)

```java
package {base-package}.application.{domain}.port.in;

import {base-package}.application.{domain}.port.in.dto.Create{Entity}InputData;
import {base-package}.application.{domain}.port.in.dto.{Entity}OutputData;

/**
 * Input Boundary — interface that defines the use case contract.
 * Controllers (adapter layer) depend on this interface, never on the Interactor.
 */
public interface Create{Entity}InputPort {

    {Entity}OutputData execute(Create{Entity}InputData input);
}
```

### 3. Input/Output Data DTOs (application/{domain}/port/in/dto)

```java
// Input Data — crossing the boundary from outside to inside
package {base-package}.application.{domain}.port.in.dto;

public record Create{Entity}InputData(String name, String description) {
    public Create{Entity}InputData {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name is required");
    }
}

// Output Data — use case result
package {base-package}.application.{domain}.port.in.dto;

import java.time.LocalDateTime;

public record {Entity}OutputData(
    String id,
    String name,
    String status,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

### 4. Output Boundary / Gateway Interface (application/{domain}/port/out)

```java
package {base-package}.application.{domain}.port.out;

import {base-package}.enterprise.{domain}.{Entity};
import {base-package}.enterprise.{domain}.{Entity}.{Entity}Id;
import java.util.List;
import java.util.Optional;

/**
 * Output Boundary / Gateway — interface defined by the Use Case.
 * Implemented by the Interface Adapters layer (Gateway Impl).
 * The Use Case never knows how data is persisted.
 */
public interface {Entity}Gateway {

    {Entity} save({Entity} entity);

    Optional<{Entity}> findById({Entity}Id id);

    List<{Entity}> findAll();

    boolean exists({Entity}Id id);

    void remove({Entity}Id id);
}
```

### 5. Use Case Interactor (application/{domain}/interactor)

```java
package {base-package}.application.{domain}.interactor;

import {base-package}.application.{domain}.port.in.Create{Entity}InputPort;
import {base-package}.application.{domain}.port.in.dto.Create{Entity}InputData;
import {base-package}.application.{domain}.port.in.dto.{Entity}OutputData;
import {base-package}.application.{domain}.port.out.{Entity}Gateway;
import {base-package}.enterprise.{domain}.{Entity};
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Use Case Interactor — implements the Input Boundary.
 * Orchestrates Entities and Gateways to execute a use case.
 * Never knows about Controllers, HTTP, JPA, or any external framework.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class Create{Entity}Interactor implements Create{Entity}InputPort {

    private final {Entity}Gateway {entity}Gateway;

    @Override
    @Transactional
    public {Entity}OutputData execute(Create{Entity}InputData input) {
        log.info("Executing Create{Entity} use case for: {}", input.name());

        var entity = {Entity}.create(input.name());
        var saved = {entity}Gateway.save(entity);

        log.info("{Entity} created successfully: {}", saved.getId());
        return toOutputData(saved);
    }

    private {Entity}OutputData toOutputData({Entity} entity) {
        return new {Entity}OutputData(
            entity.getId().toString(),
            entity.getName(),
            entity.getStatus().name(),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }
}
```

### 6. REST Controller — Interface Adapter (adapter/controller)

```java
package {base-package}.adapter.controller;

import {base-package}.adapter.controller.request.Create{Entity}Request;
import {base-package}.adapter.controller.response.{Entity}Response;
import {base-package}.application.{domain}.port.in.Create{Entity}InputPort;
import {base-package}.application.{domain}.port.in.dto.Create{Entity}InputData;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.net.URI;

/**
 * Interface Adapter — Controller.
 * Converts HTTP Request → InputData → calls Use Case → converts OutputData → HTTP Response.
 * Contains no business rules.
 */
@RestController
@RequestMapping("/api/v1/{entities}")
@RequiredArgsConstructor
public class {Entity}Controller {

    private final Create{Entity}InputPort create{Entity}InputPort;

    @PostMapping
    public ResponseEntity<{Entity}Response> create(@Valid @RequestBody Create{Entity}Request request) {
        var inputData = new Create{Entity}InputData(request.name(), request.description());
        var outputData = create{Entity}InputPort.execute(inputData);

        return ResponseEntity
            .created(URI.create("/api/v1/{entities}/" + outputData.id()))
            .body({Entity}Response.from(outputData));
    }

    @GetMapping("/{id}")
    public ResponseEntity<{Entity}Response> getById(@PathVariable String id) {
        // Similar pattern — use Get{Entity}InputPort
        return ResponseEntity.ok().build();
    }
}
```

### 7. Gateway Implementation — Interface Adapter (adapter/gateway)

```java
package {base-package}.adapter.gateway;

import {base-package}.adapter.gateway.dataaccess.{Entity}JpaRepository;
import {base-package}.adapter.gateway.mapper.{Entity}DataMapper;
import {base-package}.application.{domain}.port.out.{Entity}Gateway;
import {base-package}.enterprise.{domain}.{Entity};
import {base-package}.enterprise.{domain}.{Entity}.{Entity}Id;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.Optional;

/**
 * Gateway Implementation — concretizes the Output Boundary.
 * Translates between Enterprise Entities and Data Models.
 */
@Component
@RequiredArgsConstructor
public class {Entity}GatewayImpl implements {Entity}Gateway {

    private final {Entity}JpaRepository jpaRepository;
    private final {Entity}DataMapper mapper;

    @Override
    public {Entity} save({Entity} entity) {
        var dataModel = mapper.toDataModel(entity);
        var saved = jpaRepository.save(dataModel);
        return mapper.toEntity(saved);
    }

    @Override
    public Optional<{Entity}> findById({Entity}Id id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toEntity);
    }

    @Override
    public List<{Entity}> findAll() {
        return jpaRepository.findAll()
            .stream()
            .map(mapper::toEntity)
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

### 8. Data Model — Infrastructure (infrastructure/persistence/entity)

```java
package {base-package}.infrastructure.persistence.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Data Model / Data Access Object.
 * Belongs to the Frameworks & Drivers layer.
 * Completely decoupled from Enterprise Entities.
 */
@Entity
@Table(name = "{entities}")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class {Entity}DataModel {

    @Id
    private UUID id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(length = 500)
    private String description;

    @Column(nullable = false)
    private String status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @PrePersist
    void onPersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

### 9. Data Mapper (adapter/gateway/mapper)

```java
package {base-package}.adapter.gateway.mapper;

import {base-package}.enterprise.{domain}.{Entity};
import {base-package}.enterprise.{domain}.{Entity}.{Entity}Id;
import {base-package}.enterprise.{domain}.{Entity}Status;
import {base-package}.infrastructure.persistence.entity.{Entity}DataModel;
import org.springframework.stereotype.Component;

@Component
public class {Entity}DataMapper {

    public {Entity}DataModel toDataModel({Entity} entity) {
        return {Entity}DataModel.builder()
            .id(entity.getId().value())
            .name(entity.getName())
            .status(entity.getStatus().name())
            .build();
    }

    public {Entity} toEntity({Entity}DataModel model) {
        return {Entity}.restore(
            new {Entity}Id(model.getId()),
            model.getName(),
            {Entity}Status.valueOf(model.getStatus()),
            model.getCreatedAt(),
            model.getUpdatedAt()
        );
    }
}
```

---

## 🧪 Testing Strategy

### Test Layer by Layer

```
enterprise/     → Pure JUnit — no mocks, no Spring
application/    → JUnit + Mockito — mocked Gateway
adapter/        → @WebMvcTest (controller), @DataJpaTest (gateway)
integration/    → @SpringBootTest + Testcontainers
```

### Enterprise Entity Test (pure)

```java
class {Entity}Test {

    @Test
    void shouldCreateEntityWithActiveStatus() {
        var entity = {Entity}.create("Test Name");
        assertThat(entity.getStatus()).isEqualTo({Entity}Status.ACTIVE);
        assertThat(entity.getId()).isNotNull();
    }

    @Test
    void shouldDeactivateActiveEntity() {
        var entity = {Entity}.create("Test");
        entity.deactivate();
        assertThat(entity.getStatus()).isEqualTo({Entity}Status.INACTIVE);
    }

    @Test
    void shouldThrowWhenDeactivatingInactiveEntity() {
        var entity = {Entity}.create("Test");
        entity.deactivate();
        assertThatThrownBy(entity::deactivate)
            .isInstanceOf(IllegalStateException.class);
    }
}
```

### Interactor Test (Use Case)

```java
@ExtendWith(MockitoExtension.class)
class Create{Entity}InteractorTest {

    @Mock {Entity}Gateway gateway;
    @InjectMocks Create{Entity}Interactor interactor;

    @Test
    void shouldExecuteCreate{Entity}UseCase() {
        var input = new Create{Entity}InputData("Test", "Description");
        when(gateway.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var output = interactor.execute(input);

        assertThat(output.name()).isEqualTo("Test");
        assertThat(output.status()).isEqualTo("ACTIVE");
        verify(gateway).save(any({Entity}.class));
    }
}
```

---

## 📦 Recommended Configuration (application.yml)

```yaml
spring:
  application:
    name: {project-name}
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:{project-name}}
    username: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
  error:
    include-message: always
    include-binding-errors: always

logging:
  level:
    {base-package}: DEBUG
    org.springframework.web: INFO
```

---

## ✅ Project Generation Checklist

- [ ] Enterprise Entities with no framework annotations (zero external imports)
- [ ] Input Boundaries (interfaces) defined in `application/port/in/`
- [ ] Output Boundaries (interfaces) defined in `application/port/out/`
- [ ] Interactors depend only on Entities and Output Boundaries
- [ ] Controllers depend only on Input Boundaries (never on Interactors directly)
- [ ] Gateway Implementations are in `adapter/gateway/`, not in `infrastructure/`
- [ ] Data Models (JPA @Entity) are in `infrastructure/persistence/entity/`
- [ ] Mappers exist at each layer boundary
- [ ] Enterprise tests run without Spring and without Mockito
- [ ] Interactor tests use only Mockito, without Spring Context
- [ ] `GlobalExceptionHandler` in `infrastructure/web/exception/`
- [ ] Dependency injection via `@Bean` in `ApplicationConfig.java`

---

## 🚨 Common Mistakes — Never Do This

```java
// ❌ WRONG: Enterprise Entity knows JPA
@Entity  // NEVER in enterprise/
public class {Entity} {}

// ❌ WRONG: Interactor knows JpaRepository
@Service
public class Create{Entity}Interactor {
    @Autowired {Entity}JpaRepository repo;  // violates the Dependency Rule
}

// ❌ WRONG: Controller knows the Interactor concretely
public class {Entity}Controller {
    @Autowired Create{Entity}Interactor interactor;  // should be Create{Entity}InputPort
}

// ❌ WRONG: Gateway knows Enterprise Entity in a circular way
public class {Entity}GatewayImpl {
    // Right: adapter implements application port
    // Wrong: create domain entity directly with new without factory
}

// ✅ CORRECT: Dependency Rule respected in each layer
// enterprise → nothing
// application → enterprise
// adapter → application + enterprise
// infrastructure → everything (configurations, beans)
```
