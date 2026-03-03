# 🤖 AI Commons

> A curated collection of AI instructions, prompt templates, architecture decision guides, and project setup patterns — available in **English** and **Português (BR)**.

---

## 📌 Purpose

**AI Framework** is a structured knowledge base designed to help developers and architects harness AI tools more effectively in software engineering contexts. It provides:

- 🧠 **AI Instructions** — Ready-to-use prompts and instructions for AI assistants to generate production-ready code following best practices
- 📐 **Architecture Guidance** — Decision frameworks for choosing among Clean Architecture, Hexagonal Architecture, and Onion Architecture
- 🚀 **Project Setup Patterns** — Step-by-step scaffolding templates for Spring Boot + Java 21+ projects
- 🗺️ **Decision Guidance** — Comparison tables, trade-off analysis, and criteria-based selection guides

Whether you're starting a new microservice or evaluating architectural patterns, this framework gives you the right AI prompt to get you there faster and with higher quality.

---

## 🗂️ Repository Structure

```
ai-framework/
├── english/                          # Content in English
│   └── architecture/
│       └── code/                     # Code architecture instructions & guides
│           ├── hexagonal.md          # Hexagonal Architecture (Ports & Adapters)
│           ├── clean arch.md         # Clean Architecture (Uncle Bob)
│           ├── onion arch.md         # Onion Architecture (DDD / Jeffrey Palermo)
│           └── Java Architecture Decision Guide.md
│
└── pt-br/                            # Conteúdo em Português (BR)
    └── arquitetura/
        └── codigo/                   # Instruções e guias de arquitetura de código
            ├── hexagonal.md          # Arquitetura Hexagonal (Ports & Adapters)
            ├── clean arch.md         # Clean Architecture (Uncle Bob)
            ├── onion arch.md         # Onion Architecture (DDD / Jeffrey Palermo)
            └── Java Architecture Decision Guide.md
```

---

## 🏛️ Architecture Guides

This framework currently covers three major Java software architecture styles for Spring Boot applications. Each guide includes:

- ✅ Context and fundamental principles
- ✅ Full package structure with placeholders
- ✅ Layer-by-layer breakdown
- ✅ Code generation instructions for AI assistants
- ✅ Trade-off analysis and ideal use cases

### 🔷 Hexagonal Architecture (Ports & Adapters)
> *Alistair Cockburn, 2005*

The application core (domain + use cases) is completely isolated from the external world. All input/output goes through **Ports (interfaces)** and **Adapters (implementations)**.

**Best for:** Microservices, APIs with multiple entry points, systems needing technology flexibility.

| Layer | Description |
|-------|-------------|
| `core/domain` | Pure business entities and rules |
| `core/port/in` | Primary ports — use case contracts |
| `core/port/out` | Secondary ports — repository/external contracts |
| `core/usecase` | Use case implementations |
| `adapter/in` | REST controllers, Kafka consumers, schedulers |
| `adapter/out` | JPA persistence, HTTP clients |

---

### 🔶 Clean Architecture
> *Robert C. Martin "Uncle Bob", 2012*

Dependencies always point inward. The **Dependency Rule** is absolute: inner layers never know about outer layers.

**Best for:** Complex domain systems, large teams, long-lived enterprise applications.

| Layer | Description |
|-------|-------------|
| `enterprise/` | Entities — core business rules |
| `application/` | Use Cases / Interactors |
| `adapter/` | Controllers, Presenters, Gateways |
| `infrastructure/` | Frameworks & Drivers (JPA, Web, etc.) |

---

### 🔵 Onion Architecture
> *Jeffrey Palermo — DDD-aligned*

The **Domain Model** is the central nucleus and depends on nothing. Layers grow outward like an onion, always pointing inward. Strongly aligned with **DDD (Domain-Driven Design)** principles.

**Best for:** Domain-driven systems, medium-to-long term projects, teams practicing DDD.

| Layer | Description |
|-------|-------------|
| `domain/` | Aggregates, Entities, Value Objects, Domain Events |
| `application/` | Application Services, Commands, Queries |
| `infrastructure/` | JPA, REST, Kafka, HTTP clients |

---

## 🗺️ Architecture Decision Guide

Not sure which architecture to pick? The **Java Architecture Decision Guide** helps you make an informed choice based on:

- System complexity and domain richness
- Team size and experience
- Integration requirements (number of adapters/channels)
- Long-term maintainability needs

### Quick Comparison

| Architecture | Primary Focus | Complexity | Ideal For |
|---|---|---|---|
| **Hexagonal** | I/O isolation & integrations | Medium | APIs, microservices, multiple data sources |
| **Clean** | Business rules as the core | High | Complex domain, large teams |
| **Onion** | Concentric layers + DDD | High | Domain-driven, medium/long-term systems |

---

## 🌍 Language Support

All content is available in two languages:

| Language | Path |
|----------|------|
| 🇺🇸 English | [`/english`](./english) |
| 🇧🇷 Português (BR) | [`/pt-br`](./pt-br) |

---

## 🚀 How to Use

1. **Browse** to the language folder of your choice (`english/` or `pt-br/`)
2. **Navigate** to the relevant category (e.g., `architecture/code/`)
3. **Open** the guide for the architecture you want to use or evaluate
4. **Copy** the AI instruction prompt into your AI assistant (GitHub Copilot, ChatGPT, Claude, etc.)
5. **Provide** the required placeholders (e.g., `{base-package}`, `{Entity}`, `{UseCase}`)
6. **Generate** a fully structured, production-ready Spring Boot project

### Example — Using Hexagonal Architecture Prompt

```
1. Open: english/architecture/code/hexagonal.md
2. Copy the full AI instruction
3. Paste into GitHub Copilot Chat or your AI tool
4. Provide inputs:
   - base-package: com.mycompany.myservice
   - Entity: Order
   - UseCase: CreateOrder
5. The AI will generate a complete, layered project structure
```

---

## 🛣️ Roadmap

> Content planned for future additions:

- [ ] 🐳 Docker & containerization setup guides
- [ ] ☁️ Cloud-native project patterns (AWS, GCP, Azure)
- [ ] 🔐 Security patterns (OAuth2, JWT with Spring Security)
- [ ] 📊 Observability setup (OpenTelemetry, Micrometer, Grafana)
- [ ] 🧪 Testing strategy guides (Unit, Integration, Contract)
- [ ] 🔄 Event-driven architecture patterns (Kafka, RabbitMQ)
- [ ] 🌐 API design guidelines (REST, gRPC, GraphQL)

---

## 🤝 Contributing

Contributions are welcome! If you have a great AI instruction, architecture pattern, or project setup guide you'd like to share:

1. Fork the repository
2. Create a branch: `git checkout -b feature/my-new-guide`
3. Add your content following the existing structure
4. Submit a Pull Request with a clear description

Please ensure content is available in **both English and Portuguese (BR)** when possible.

---

## 📄 License

This project is open source. Feel free to use, adapt, and share the content within your teams and projects.

---

<div align="center">

Made with ❤️ to help developers build better software with AI

⭐ Star this repo if you find it useful!

</div>
