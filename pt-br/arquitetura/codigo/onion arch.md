# 🔵 INSTRUÇÃO DE IA — Onion Architecture
> Spring Boot · Java 21+ · Maven/Gradle · DDD · Jeffrey Palermo

---

## 🎯 Contexto da Arquitetura

Você é um especialista em arquitetura de software Java com Spring Boot e Domain-Driven Design. Sua tarefa é criar um projeto Spring Boot completo seguindo rigorosamente a **Onion Architecture** de Jeffrey Palermo, fortemente alinhada com os princípios de **DDD (Domain-Driven Design)**.

**Princípio fundamental:** O **Domain Model** é o núcleo central e não depende de nada. As camadas crescem ao redor como as camadas de uma cebola, sempre apontando para dentro. A infraestrutura fica na camada mais externa e é completamente substituível.

**Camadas (de dentro para fora):**
1. **Domain Model** — Entidades, Value Objects, Aggregates, Domain Events, Enums
2. **Domain Services** — Serviços que operam sobre o Domain Model (stateless, sem infra)
3. **Application Services** — Orquestra Domain Services e Repositories (casos de uso)
4. **Infrastructure / UI** — JPA, REST, Kafka, HTTP clients, tudo externo

**Diferença chave em relação a Clean:** O Onion usa terminologia e conceitos de DDD explicitamente (Aggregates, Value Objects, Repositories como interfaces de domínio, Domain Events).

---

## 📐 Estrutura de Pacotes

```
src/
└── main/
    └── java/
        └── {base-package}/
            │
            ├── domain/                                ← CAMADA 1: DOMAIN MODEL
            │   └── {boundedcontext}/
            │       ├── model/
            │       │   ├── {Aggregate}.java           ← Aggregate Root
            │       │   ├── {Entity}.java              ← Entity (dentro do aggregate)
            │       │   └── valueobject/
            │       │       ├── {Entity}Id.java        ← Value Object - ID
            │       │       ├── Money.java             ← Value Object - exemplo
            │       │       └── Email.java             ← Value Object - exemplo
            │       ├── event/
            │       │   └── {Entity}CreatedEvent.java  ← Domain Event
            │       ├── repository/
            │       │   └── {Aggregate}Repository.java ← Repository Interface (domínio)
            │       ├── service/
            │       │   └── {Entity}DomainService.java ← Domain Service
            │       └── exception/
            │           ├── {Entity}NotFoundException.java
            │           └── {Entity}BusinessException.java
            │
            ├── application/                           ← CAMADA 2: APPLICATION SERVICES
            │   └── {boundedcontext}/
            │       ├── service/
            │       │   └── {UseCase}ApplicationService.java
            │       └── dto/
            │           ├── command/
            │           │   └── Create{Entity}Command.java
            │           └── query/
            │               └── {Entity}QueryResult.java
            │
            └── infrastructure/                        ← CAMADA 3: INFRASTRUCTURE
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
            ├── domain/                                ← JUnit puro — domínio isolado
            ├── application/                           ← JUnit + Mockito
            └── infrastructure/
                ├── persistence/                       ← @DataJpaTest
                ├── web/                               ← @WebMvcTest
                └── integration/                       ← @SpringBootTest + Testcontainers
```

---

## 📋 Regras de Dependência (OBRIGATÓRIO)

```
[Domain Model]         ← zero dependências externas
      ↑
[Domain Services]      ← depende apenas de Domain Model
      ↑
[Application Services] ← depende de Domain + Domain Services
      ↑
[Infrastructure]       ← depende de tudo, implementa interfaces de domínio
```

**Regras DDD absolutas — nunca viole:**
1. `domain/model/` tem **zero dependências externas** — Java puro
2. `domain/repository/` são interfaces — nunca dependem de JPA ou Spring
3. `domain/service/` não acessa bancos de dados diretamente
4. `application/service/` não acessa JPA ou infraestrutura diretamente
5. `infrastructure/` implementa os contratos definidos em `domain/`
6. Aggregates encapsulam invariantes — nunca exponha coleções internas mutáveis

---

## 🗂️ DDD Building Blocks & Templates

### 1. Value Object (domain/{boundedcontext}/model/valueobject)

```java
package {base-package}.domain.{boundedcontext}.model.valueobject;

import java.util.Objects;
import java.util.UUID;

/**
 * Value Object — imutável, sem identidade, igualdade por valor.
 * Use records para VOs simples (Java 16+).
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

// Exemplo de Value Object com regra de negócio
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
 * Aggregate Root — ponto de entrada para o aggregate.
 * Protege invariantes e emite Domain Events.
 * Entidades filhas só são acessadas via o Aggregate Root.
 */
public class {Entity} {

    private final {Entity}Id id;
    private String name;
    private {Entity}Status status;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Domain Events — acumulados para publicação após persistência
    private final List<Object> domainEvents = new ArrayList<>();

    // Construtor privado — use factory methods
    private {Entity}({Entity}Id id, String name) {
        this.id = id;
        this.name = name;
        this.status = {Entity}Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    /**
     * Factory Method — criação com validação de invariantes.
     */
    public static {Entity} create(String name) {
        validateName(name);
        var entity = new {Entity}({Entity}Id.generate(), name);
        entity.registerEvent(new {Entity}CreatedEvent(entity.id, entity.name, entity.createdAt));
        return entity;
    }

    /**
     * Reconstituição — usado por repositórios para restaurar estado.
     */
    public static {Entity} reconstitute({Entity}Id id, String name, {Entity}Status status,
                                        LocalDateTime createdAt, LocalDateTime updatedAt) {
        var entity = new {Entity}(id, name);
        entity.status = status;
        entity.updatedAt = updatedAt;
        return entity;
    }

    // Comportamentos de domínio (métodos expressivos — Ubiquitous Language)
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

    // Retorna cópia imutável — não expõe lista interna mutável
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
 * Domain Event — representa algo que aconteceu no domínio.
 * Imutável, nomeado no passado.
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
 * Repository — interface definida no domínio.
 * Linguagem do domínio (não "save/find" genérico, mas métodos expressivos).
 * Implementada pela infraestrutura.
 */
public interface {Entity}Repository {

    void store({Entity} {entity});          // Preferir nomes de domínio

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
 * Domain Service — lógica de domínio que não pertence a uma única entidade.
 * Stateless. Pode usar Repository interfaces do domínio.
 * NÃO orquestra casos de uso — isso é papel do Application Service.
 */
@Service
@RequiredArgsConstructor
public class {Entity}DomainService {

    private final {Entity}Repository {entity}Repository;

    /**
     * Verifica se o nome já existe no domínio (regra de unicidade).
     */
    public boolean isNameUnique(String name) {
        // Implementação com query no repository
        return true; // exemplo simplificado
    }

    /**
     * Exemplo: lógica complexa que envolve múltiplas entidades.
     */
    public void validateBusinessRule({Entity} {entity}) {
        // Regra de negócio que precisa de contexto além da própria entidade
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
 * Application Service — orquestra o fluxo de um caso de uso.
 * Coordena Domain Services e Repositories.
 * Publica Domain Events após transação.
 * NÃO contém regras de negócio — delega ao domínio.
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

        // Validação de regra de negócio via Domain Service
        if (!{entity}DomainService.isNameUnique(command.name())) {
            throw new IllegalArgumentException("{Entity} name already exists: " + command.name());
        }

        // Criação via factory method do Aggregate
        var {entity} = {Entity}.create(command.name());

        // Persistência via Repository do domínio
        {entity}Repository.store({entity});

        // Publicação de Domain Events
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
// Command — intenção de mudança de estado
package {base-package}.application.{boundedcontext}.dto.command;

public record Create{Entity}Command(String name, String description) {
    public Create{Entity}Command {
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name is required");
    }
}

// Query Result — resultado de leitura
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
 * ORM Entity — detalhe de persistência, não é o Domain Model.
 * Usa nomenclatura diferente ({Entity}OrmEntity) para não confundir.
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
    private Long version;  // Optimistic locking — DDD recomenda

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
 * Repository Implementation — na infraestrutura, implementa interface do domínio.
 * Traduz entre Domain Aggregate ↔ ORM Entity.
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
 * REST Controller — infraestrutura de UI.
 * Converte HTTP ↔ Application Commands/Queries.
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
 * Domain Event Handler — processa Domain Events após commit da transação.
 * Pode publicar para Kafka, enviar emails, notificar outros bounded contexts.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class {Entity}EventPublisher {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on{Entity}Created({Entity}CreatedEvent event) {
        log.info("Handling domain event: {Entity}Created for id={}", event.{entity}Id());
        // Publicar para Kafka, webhook, etc.
    }
}
```

---

## 🧪 Estratégia de Testes com DDD

### Teste do Domain Model (JUnit puro — zero mocks)

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

### Teste do Application Service (Mockito)

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

## 📦 Dependências DDD Recomendadas (pom.xml)

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

## ✅ Checklist DDD + Onion — Geração do Projeto

- [ ] Domain Model sem nenhuma anotação de framework ou imports externos
- [ ] Value Objects são imutáveis (records ou final fields + no setters)
- [ ] Aggregate Root é o único ponto de entrada para o Aggregate (setter privado)
- [ ] Domain Events criados como records imutáveis e nomeados no passado
- [ ] `pullDomainEvents()` limpa eventos após retorná-los
- [ ] Repository interfaces em `domain/` usam linguagem ubíqua
- [ ] Domain Services não acessam infraestrutura diretamente
- [ ] Application Services orquestram — não contêm regras de domínio
- [ ] `@TransactionalEventListener(phase = AFTER_COMMIT)` para Domain Events
- [ ] ORM Entities nomeadas diferentemente das Domain Entities (ex: `OrmEntity`)
- [ ] Mappers entre Domain ↔ ORM isolados em `infrastructure/persistence/mapper/`
- [ ] Testes do Domain rodando sem Spring Context
- [ ] Testes do Application Service com Mockito (sem Spring)
- [ ] Testes de persistência com `@DataJpaTest` + H2/Testcontainers
- [ ] Bounded Contexts bem definidos em `domain/` (subpacotes por contexto)

---

## 🚨 Erros Comuns — Nunca Faça

```java
// ❌ ERRADO: Aggregate com JPA annotation
@Entity  // JAMAIS no domain/model
public class {Entity} {}

// ❌ ERRADO: Domain Service acessa infraestrutura
@Service
public class {Entity}DomainService {
    @Autowired EntityManager em;  // viola o isolamento do domínio
}

// ❌ ERRADO: Application Service contém regra de negócio
@Service
public class {Entity}ApplicationService {
    public void create{Entity}(Create{Entity}Command cmd) {
        if (cmd.name().length() > 100) throw ...;  // isso é regra de domínio!
    }
}

// ❌ ERRADO: Controller acessa Domain Service diretamente
@RestController
public class {Entity}Controller {
    @Autowired {Entity}DomainService domainService;  // controller só acessa Application Service
}

// ❌ ERRADO: Repository de domínio estende JpaRepository
public interface {Entity}Repository extends JpaRepository<...> {}  // domínio não pode depender de JPA

// ✅ CORRETO: Repository de domínio é interface pura
public interface {Entity}Repository {
    void store({Entity} {entity});
    Optional<{Entity}> findById({Entity}Id id);
}

// ✅ CORRETO: Domain Event publicado após commit
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void on{Entity}Created({Entity}CreatedEvent event) { ... }
```

---

## 🗣️ Glossário DDD (Ubiquitous Language)

| Termo | Significado | Implementação |
|---|---|---|
| **Aggregate** | Cluster de entidades tratadas como unidade | Classe com factory method |
| **Aggregate Root** | Ponto de entrada do Aggregate | Entidade principal com `create()` |
| **Value Object** | Objeto sem identidade, igual por valor | `record` Java |
| **Domain Event** | Algo que aconteceu no domínio | `record` nomeado no passado |
| **Repository** | Abstração de coleção de Aggregates | Interface no domínio |
| **Domain Service** | Lógica que não pertence a uma entidade | `@Service` no domínio |
| **Application Service** | Orquestra casos de uso | `@Service` em application/ |
| **Bounded Context** | Fronteira de um subdomínio | Subpacote em domain/ |
| **Ubiquitous Language** | Linguagem compartilhada com especialistas | Nomes dos métodos/classes |
