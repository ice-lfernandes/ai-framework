# 🏗️ Java Architecture Decision Guide
> AI-powered decision framework for choosing between Hexagonal, Clean, and Onion Architecture

---

## 📋 Overview

This guide helps Java architects and developers choose the most suitable code architecture for their Spring Boot projects, based on system characteristics, team profile, and business context.

### Available Architectures

| Architecture | Main Focus | Complexity | Ideal For |
|---|---|---|---|
| **Hexagonal (Ports & Adapters)** | I/O and integration isolation | Medium | APIs, microservices, multiple data sources |
| **Clean Architecture** | Business rules as the core | High | Complex domain systems, large teams |
| **Onion Architecture** | Concentric layers, DDD | High | Domain-driven, medium/long-term systems |

---

## 🔍 Research: Architecture Overview

### 🔷 Hexagonal Architecture (Ports & Adapters)
**Creator:** Alistair Cockburn (2005)

**Terminology:**
- **Core / Application** — pure business logic
- **Ports** — interfaces that define input/output contracts
  - *Primary/Driving Ports* — use cases triggered by external actors (e.g., REST, CLI)
  - *Secondary/Driven Ports* — contracts for external dependencies (e.g., DB, messaging)
- **Adapters** — concrete implementations of ports
  - *Primary Adapters* — REST controllers, Kafka consumers, CLI handlers
  - *Secondary Adapters* — JPA repositories, HTTP clients, event publishers

**Advantages:**
- Extreme testability: core independent of frameworks
- Easy technology swap (replace JPA with MongoDB without business impact)
- Clear separation between "what" (core) and "how" (adapters)
- Excellent for multiple input channels (REST + gRPC + Kafka)

**Disadvantages:**
- Proliferation of interfaces and mapping classes
- Learning curve for teams used to MVC
- Can be over-engineering for simple applications
- Mapping between layers (DTO ↔ Domain ↔ Entity) generates boilerplate

**Ideal Use Cases:**
- Microservices with multiple input/output adapters
- Systems that need to swap persistence technologies
- APIs serving multiple protocols
- Applications with moderate business logic

---

### 🔶 Clean Architecture
**Creator:** Robert C. Martin "Uncle Bob" (2012)

**Terminology:**
- **Entities** — corporate business rules, independent of any application
- **Use Cases (Interactors)** — application business rules, orchestrate Entities
- **Interface Adapters** — Controllers, Presenters, Gateways — convert data
- **Frameworks & Drivers** — DB, Web, UI, external devices
- **Dependency Rule** — dependencies always point inward (toward the rules)

**Advantages:**
- Complete independence from frameworks, UI, databases, and external agents
- Testability across all layers without complex mocks
- Business rules protected and stable against external changes
- Scales well in large teams (well-defined domains)

**Disadvantages:**
- High structural complexity, many files and abstractions
- Longer initial setup time
- Can be excessive for simple domains or pure CRUD
- DTOs and mappings between layers multiply
- Steeper learning curve than Hexagonal

**Ideal Use Cases:**
- Enterprise systems with highly complex business rules
- Large teams needing clear boundaries between sub-teams
- Long-term products with constant domain evolution
- Systems where business logic is the most critical asset

---

### 🔵 Onion Architecture
**Creator:** Jeffrey Palermo (2008)

**Terminology:**
- **Domain Model** — core: entities, value objects, enums, domain exceptions
- **Domain Services** — services that operate on the domain model (no external dependencies)
- **Application Services** — use case orchestration, coordinates domain services
- **Infrastructure / UI** — outermost layer: persistence, API, messaging
- **Interfaces / Ports** — defined in inner layers, implemented in outer layers

**Advantages:**
- Strong alignment with DDD (Domain-Driven Design)
- Domain Model completely isolated (no external dependencies)
- Great for rich domain modeling (rich domain model)
- Encourages collaboration with business experts (Ubiquitous Language)

**Disadvantages:**
- Requires deep DDD knowledge to be well implemented
- Harder to define correct Bounded Context boundaries
- Risk of domain anemia (empty domain model) if poorly applied
- Less documentation and practical examples than Clean and Hexagonal

**Ideal Use Cases:**
- Systems with complex domain and rich business rules
- Contexts where DDD is used (Bounded Contexts, Aggregates, etc.)
- Medium/long-term applications with involved domain experts
- Financial systems, ERP, regulatory systems

---

## ❓ Decision Questionnaire

> Answer the questions below to determine which architecture to use.
> Use the AI agent with the prompt in the following section to do this automatically.

### Block 1 — Domain Complexity

**Q1. How would you describe your application's business rules?**
- A) Simple: mostly CRUD, few rules → *Hexagonal*
- B) Moderate: some specific business flows → *Hexagonal or Clean*
- C) Complex: many rules, rich entities, invariants → *Clean or Onion*
- D) Very complex: deep domain, specific business language → *Onion*

**Q2. Is a Domain Expert involved in the project?**
- A) No → *Hexagonal or Clean*
- B) Yes, occasionally → *Clean*
- C) Yes, actively (practicing DDD / Ubiquitous Language) → *Onion*

---

### Block 2 — Integrations and I/O

**Q3. How many input sources will your application have?**
- A) Only one (e.g., REST API only) → *any*
- B) Two or more (e.g., REST + Kafka + CLI) → *Hexagonal*
- C) Many and varied (REST, gRPC, Kafka, WebSocket) → *Hexagonal (strong indication)*

**Q4. How many external systems or databases does your application integrate?**
- A) One or two → *any*
- B) Three or more → *Hexagonal*
- C) Planning to switch database/broker in the future → *Hexagonal*

---

### Block 3 — Team and Context

**Q5. What is your team size?**
- A) 1–3 developers → *Hexagonal*
- B) 4–8 developers → *Clean or Hexagonal*
- C) More than 8 developers / multiple teams → *Clean or Onion*

**Q6. What is the average experience level of the team with architecture?**
- A) Junior/Mid, no experience with advanced architectures → *Hexagonal*
- B) Senior, familiar with Clean Code and SOLID → *Clean*
- C) Senior+, with DDD experience → *Onion*

**Q7. What is the expected lifespan of the project?**
- A) Short term (< 1 year, POC, MVP) → *Hexagonal*
- B) Medium term (1–3 years) → *Clean or Hexagonal*
- C) Long term (> 3 years, core business product) → *Clean or Onion*

---

### Block 4 — Testability and Quality

**Q8. How important are automated tests to you?**
- A) Low: only basic integration tests → *any*
- B) Medium: unit and integration tests → *Hexagonal*
- C) High: high coverage, TDD, isolated domain tests → *Clean or Onion*

**Q9. Do you need to swap technologies easily (e.g., replace JPA with MongoDB)?**
- A) No → *any*
- B) Maybe → *Hexagonal*
- C) Yes, it's a requirement → *Hexagonal (strong indication)*

---

## 🤖 AI Prompt for Automatic Decision

Paste the prompt below into your preferred AI agent (Claude, GPT-4, etc.) to get a personalized recommendation:

```
You are an expert in Java and Spring Boot software architecture.
I will answer a questionnaire and you should recommend the best architecture
among: Hexagonal (Ports & Adapters), Clean Architecture, or Onion Architecture.

At the end, justify your choice based on my answers and provide me with:
1. The recommended architecture
2. Justification based on my answers
3. Specific points of attention for my context
4. Which instruction file to use: HEXAGONAL_ARCH.md, CLEAN_ARCH.md or ONION_ARCH.md

Here are my answers:

Q1 - Domain complexity: [your answer]
Q2 - Domain expert involved: [your answer]
Q3 - Input sources: [your answer]
Q4 - External integrations: [your answer]
Q5 - Team size: [your answer]
Q6 - Experience level: [your answer]
Q7 - Project lifespan: [your answer]
Q8 - Importance of tests: [your answer]
Q9 - Need to swap technologies: [your answer]
```

---

## 📁 Instruction Files Structure

```
java-arch-ai-instructions/
├── README.md                  ← This file (decision guide)
├── HEXAGONAL_ARCH.md          ← Instruction for Hexagonal Architecture
├── CLEAN_ARCH.md              ← Instruction for Clean Architecture
└── ONION_ARCH.md              ← Instruction for Onion Architecture
```

---

## ⚡ Quick Decision

| If your project is... | Use |
|---|---|
| Microservice with multiple adapters (REST + Kafka + DB) | **HEXAGONAL_ARCH.md** |
| Simple API with some business logic | **HEXAGONAL_ARCH.md** |
| Enterprise system with complex domain, large team | **CLEAN_ARCH.md** |
| DDD applied, rich domain, business experts | **ONION_ARCH.md** |
| POC / MVP / short term | **HEXAGONAL_ARCH.md** |
| Strategic long-term product | **CLEAN_ARCH.md** or **ONION_ARCH.md** |
