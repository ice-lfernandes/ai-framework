# 🔷 INSTRUÇÃO DE IA — Hexagonal Architecture (Ports & Adapters)
> Spring Boot · Java 21+ · Maven/Gradle

---

## 🎯 Contexto da Arquitetura

Você é um especialista em arquitetura de software Java com Spring Boot. Sua tarefa é criar um projeto Spring Boot completo seguindo rigorosamente a **Hexagonal Architecture (Ports & Adapters)**.

**Princípio fundamental:** O core da aplicação (domínio + casos de uso) é completamente isolado do mundo externo. Todo acesso de entrada e saída ocorre via **Portas (interfaces)** e **Adaptadores (implementações)**.

---

## 📐 Estrutura de Pacotes

Gere o projeto com a seguinte estrutura de diretórios. Use `{base-package}` como placeholder para o pacote base informado pelo usuário:

```
src/
└── main/
    └── java/
        └── {base-package}/
            ├── core/                                  ← NÚCLEO DA APLICAÇÃO
            │   ├── domain/                            ← Entidades e regras puras
            │   │   ├── model/                         ← Domain models (entidades, VOs)
            │   │   ├── exception/                     ← Domain exceptions
            │   │   └── event/                         ← Domain events (opcional)
            │   ├── port/                              ← CONTRATOS (interfaces)
            │   │   ├── in/                            ← Primary ports (use cases)
            │   │   │   └── {UseCase}UseCase.java
            │   │   └── out/                           ← Secondary ports (repositories, external)
            │   │       └── {Entity}Repository.java
            │   └── usecase/                           ← Implementações dos casos de uso
            │       └── {UseCase}UseCaseImpl.java
            │
            └── adapter/                               ← ADAPTADORES
                ├── in/                                ← Primary adapters (entry points)
                │   ├── rest/                          ← REST Controllers
                │   │   ├── {Entity}Controller.java
                │   │   ├── request/
                │   │   │   └── {Entity}Request.java
                │   │   └── response/
                │   │       └── {Entity}Response.java
                │   ├── consumer/                      ← Kafka/RabbitMQ consumers (se aplicável)
                │   └── scheduler/                     ← Scheduled tasks (se aplicável)
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
            │   └── usecase/                           ← Testes unitários puros (sem Spring)
            ├── adapter/
            │   ├── in/rest/                           ← @WebMvcTest por controller
            │   └── out/persistence/                   ← @DataJpaTest por adapter
            └── integration/                           ← @SpringBootTest
```

---

## 📋 Regras de Dependência (OBRIGATÓRIO)

```
adapter/in  →  core/port/in  →  core/usecase  →  core/port/out  ←  adapter/out
                                      ↓
                               core/domain/model
```

**Regras absolutas — nunca viole:**
1. `core/` **NUNCA** importa nada de `adapter/`
2. `core/domain/` **NUNCA** importa Spring, JPA ou qualquer framework
3. `core/usecase/` pode importar apenas `core/domain/` e `core/port/`
4. `adapter/in/` depende de `core/port/in/` (nunca de `usecase/` diretamente)
5. `adapter/out/` implementa `core/port/out/`
6. Mapeamentos sempre acontecem nos **adapters**, nunca no core

---

## 🗂️ Templates de Código

### 1. Domain Model (core/domain/model)

```java
package {base-package}.core.domain.model;

import java.util.UUID;
import java.time.LocalDateTime;

/**
 * Domain Entity — sem anotações de framework.
 * Contém regras de negócio e invariantes do domínio.
 */
public class {Entity} {

    private final {Entity}Id id;
    private String name;
    private {Entity}Status status;
    private final LocalDateTime createdAt;

    // Construtor privado — use factory methods
    private {Entity}(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.status = {Entity}Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
    }

    // Factory method para nova entidade
    public static {Entity} create(String name) {
        validateName(name);
        return new Builder()
            .id(new {Entity}Id(UUID.randomUUID()))
            .name(name)
            .build();
    }

    // Factory method para reconstituição (ex: do banco)
    public static {Entity} reconstitute({Entity}Id id, String name, {Entity}Status status, LocalDateTime createdAt) {
        // ...
    }

    // Regras de negócio — métodos expressivos
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

    // Getters (sem setters — imutabilidade preferida)
    public {Entity}Id getId() { return id; }
    public String getName() { return name; }
    public {Entity}Status getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    // Value Object para ID
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
 * Primary Port — define o contrato de um caso de uso.
 * Acionado por Primary Adapters (REST, Kafka consumer, etc.).
 */
public interface Create{Entity}UseCase {

    {Entity}Result execute(Create{Entity}Command command);

    /**
     * Command — dados de entrada do caso de uso (imutável).
     * Validações básicas aqui; regras complexas no domínio.
     */
    record Create{Entity}Command(String name, String description) {
        public Create{Entity}Command {
            if (name == null || name.isBlank()) {
                throw new IllegalArgumentException("Name is required");
            }
        }
    }

    /**
     * Result — dados de saída do caso de uso.
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
 * Secondary Port — contrato que o core exige para persistência.
 * Implementado por adapter/out/persistence.
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
 * Use Case Implementation — orquestra domínio e portas.
 * @Service é permitido aqui pois é framework de injeção, não de negócio.
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
 * Converte HTTP ↔ Commands/Results. Não contém lógica de negócio.
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
 * JPA Entity — pertence ao adapter, nunca ao domínio.
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
 * Mapper entre Domain Model ↔ JPA Entity.
 * Nunca use @Mapper do MapStruct direto nos ports — sempre neste adapter.
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
 * Secondary Adapter — implementa a porta de repositório.
 * Traduz entre JPA e Domain Model.
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
 * Spring Data JPA — detalhe de infraestrutura.
 * Permanece totalmente dentro do adapter/out.
 */
public interface {Entity}JpaRepository extends JpaRepository<{Entity}JpaEntity, UUID> {
    // Custom queries aqui se necessário
}
```

---

## 🧪 Estratégia de Testes

### Teste de Use Case (sem Spring — JUnit 5 + Mockito)

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

### Teste de Controller (Spring MVC slice)

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

### Teste de Persistence Adapter (JPA slice)

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

## 📦 pom.xml — Dependências Essenciais

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

## ✅ Checklist de Geração do Projeto

Ao criar o projeto, **verifique cada item antes de finalizar:**

- [ ] Domain model sem nenhuma anotação de framework (`@Entity`, `@Component`, etc.)
- [ ] Portas (`port/in` e `port/out`) são interfaces puras sem dependências de framework
- [ ] Use Cases não conhecem `@RestController`, `JpaRepository`, nem nada de `adapter/`
- [ ] Adapters fazem mapeamento — core nunca recebe DTOs ou JPA entities diretamente
- [ ] `@Service` e `@Component` existem apenas em `usecase/` e `adapter/`
- [ ] Exceções de domínio criadas em `core/domain/exception/`
- [ ] `GlobalExceptionHandler` criado em `adapter/in/rest/` (não no core)
- [ ] Testes de use case rodam **sem Spring Context** (apenas JUnit + Mockito)
- [ ] Testes de controller usam `@WebMvcTest` (não `@SpringBootTest`)
- [ ] Testes de persistência usam `@DataJpaTest` (não `@SpringBootTest`)
- [ ] Migrações Flyway criadas em `resources/db/migration/`
- [ ] `application.yml` configurado com perfis (`dev`, `test`, `prod`)

---

## 🚨 Erros Comuns — Nunca Faça

```java
// ❌ ERRADO: domain model com JPA
@Entity  // Jamais no core/domain/model
public class {Entity} { ... }

// ❌ ERRADO: use case conhece o controller
@RestController  // Jamais no core/usecase
public class Create{Entity}UseCase { ... }

// ❌ ERRADO: controller chama repository diretamente
@RestController
public class {Entity}Controller {
    @Autowired {Entity}JpaRepository jpaRepository; // bypass total da arquitetura
}

// ❌ ERRADO: porta de saída com dependência de JPA
public interface {Entity}Repository extends JpaRepository<...> {} // port/out não pode ter isso

// ✅ CORRETO: port é interface pura
public interface {Entity}Repository {
    {Entity} save({Entity} entity);
}
```
