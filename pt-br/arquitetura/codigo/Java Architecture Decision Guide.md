# 🏗️ Java Architecture Decision Guide
> AI-powered decision framework for choosing between Hexagonal, Clean, and Onion Architecture

---

## 📋 Overview

Este guia auxilia arquitetos e desenvolvedores Java a escolherem a arquitetura de código mais adequada para seus projetos Spring Boot, com base nas características do sistema, equipe e contexto de negócio.

### Arquiteturas Disponíveis

| Arquitetura | Foco Principal | Complexidade | Ideal Para |
|---|---|---|---|
| **Hexagonal (Ports & Adapters)** | Isolamento de I/O e integrações | Média | APIs, microservices, múltiplas fontes de dados |
| **Clean Architecture** | Regras de negócio como núcleo | Alta | Sistemas complexos de domínio, grandes equipes |
| **Onion Architecture** | Camadas concêntricas, DDD | Alta | Domain-driven, sistemas de médio/longo prazo |

---

## 🔍 Pesquisa: Visão Geral das Arquiteturas

### 🔷 Hexagonal Architecture (Ports & Adapters)
**Criador:** Alistair Cockburn (2005)

**Terminologia:**
- **Core / Application** — lógica de negócio pura
- **Ports (Portas)** — interfaces que definem contratos de entrada/saída
  - *Primary/Driving Ports* — casos de uso acionados por atores externos (ex: REST, CLI)
  - *Secondary/Driven Ports* — contratos para dependências externas (ex: DB, messaging)
- **Adapters (Adaptadores)** — implementações concretas das portas
  - *Primary Adapters* — controllers REST, consumers Kafka, CLI handlers
  - *Secondary Adapters* — repositórios JPA, clientes HTTP, publishers de eventos

**Vantagens:**
- Testabilidade extrema: core independente de frameworks
- Fácil troca de tecnologias (trocar JPA por MongoDB sem impacto no negócio)
- Separação clara entre "o que" (core) e "como" (adapters)
- Excelente para múltiplos canais de entrada (REST + gRPC + Kafka)

**Desvantagens:**
- Proliferação de interfaces e classes de mapeamento
- Curva de aprendizado para equipes acostumadas com MVC
- Pode ser overengineering para aplicações simples
- Mapeamento entre camadas (DTO ↔ Domain ↔ Entity) gera boilerplate

**Casos de Uso Ideais:**
- Microservices com múltiplos adaptadores de entrada/saída
- Sistemas que precisam trocar tecnologias de persistência
- APIs que servirão múltiplos protocolos
- Aplicações com lógica de negócio moderada

---

### 🔶 Clean Architecture
**Criador:** Robert C. Martin "Uncle Bob" (2012)

**Terminologia:**
- **Entities** — regras de negócio corporativas, independentes de qualquer aplicação
- **Use Cases (Interactors)** — regras de negócio da aplicação, orquestram Entities
- **Interface Adapters** — Controllers, Presenters, Gateways — convertem dados
- **Frameworks & Drivers** — DB, Web, UI, dispositivos externos
- **Dependency Rule** — dependências sempre apontam para dentro (para as regras)

**Vantagens:**
- Independência total de frameworks, UI, banco de dados e agentes externos
- Testabilidade em todas as camadas sem mocks complexos
- Regras de negócio protegidas e estáveis a mudanças externas
- Escala bem em grandes equipes (domínios bem delimitados)

**Desvantagens:**
- Alta complexidade estrutural, muitos arquivos e abstrações
- Tempo maior de setup inicial
- Pode ser excessivo para domínios simples ou CRUD puro
- DTOs e mapeamentos entre camadas se multiplicam
- Curva de aprendizado mais elevada que Hexagonal

**Casos de Uso Ideais:**
- Sistemas enterprise com regras de negócio altamente complexas
- Equipes grandes que precisam de fronteiras claras entre times
- Produtos de longo prazo com evolução constante do domínio
- Sistemas onde a lógica de negócio é o ativo mais crítico

---

### 🔵 Onion Architecture
**Criador:** Jeffrey Palermo (2008)

**Terminologia:**
- **Domain Model** — núcleo: entidades, value objects, enums, exceções de domínio
- **Domain Services** — serviços que operam sobre o modelo de domínio (sem dependências externas)
- **Application Services** — orquestração de casos de uso, coordena serviços de domínio
- **Infrastructure / UI** — camada mais externa: persistência, API, mensageria
- **Interfaces / Ports** — definidas nas camadas internas, implementadas nas externas

**Vantagens:**
- Forte alinhamento com DDD (Domain-Driven Design)
- Domain Model completamente isolado (sem nenhuma dependência externa)
- Ótima para modelagem rica de domínio (rich domain model)
- Favorece a colaboração com especialistas de negócio (Ubiquitous Language)

**Desvantagens:**
- Requer profundo conhecimento de DDD para ser bem implementada
- Maior dificuldade em definir as fronteiras corretas de Bounded Contexts
- Risco de anemia do domínio (domain model vazio) se mal aplicada
- Menos documentação e exemplos práticos que Clean e Hexagonal

**Casos de Uso Ideais:**
- Sistemas com domínio complexo e regras de negócio ricas
- Contextos onde DDD é utilizado (Bounded Contexts, Aggregates, etc.)
- Aplicações de médio/longo prazo com especialistas de domínio envolvidos
- Sistemas financeiros, ERP, sistemas regulatórios

---

## ❓ Questionário de Decisão

> Responda as perguntas abaixo para determinar qual arquitetura utilizar.
> Use o agente de IA com o prompt na seção seguinte para fazer isso de forma automática.

### Bloco 1 — Complexidade do Domínio

**P1. Como você descreveria as regras de negócio da sua aplicação?**
- A) Simples: majoritariamente CRUD, poucas regras → *Hexagonal*
- B) Moderadas: alguns fluxos de negócio específicos → *Hexagonal ou Clean*
- C) Complexas: muitas regras, entidades ricas, invariantes → *Clean ou Onion*
- D) Muito complexas: domínio profundo, linguagem de negócio específica → *Onion*

**P2. Existe um especialista de domínio (Domain Expert) envolvido no projeto?**
- A) Não → *Hexagonal ou Clean*
- B) Sim, ocasionalmente → *Clean*
- C) Sim, ativamente (praticando DDD / Ubiquitous Language) → *Onion*

---

### Bloco 2 — Integrações e I/O

**P3. Quantas fontes de entrada sua aplicação terá?**
- A) Apenas uma (ex: só REST API) → *qualquer*
- B) Duas ou mais (ex: REST + Kafka + CLI) → *Hexagonal*
- C) Muitas e variadas (REST, gRPC, Kafka, WebSocket) → *Hexagonal (forte indicação)*

**P4. Quantos sistemas externos ou bancos de dados sua aplicação integra?**
- A) Um ou dois → *qualquer*
- B) Três ou mais → *Hexagonal*
- C) Planeja trocar de banco/broker futuramente → *Hexagonal*

---

### Bloco 3 — Equipe e Contexto

**P5. Qual o tamanho da sua equipe?**
- A) 1–3 desenvolvedores → *Hexagonal*
- B) 4–8 desenvolvedores → *Clean ou Hexagonal*
- C) Mais de 8 desenvolvedores / múltiplos times → *Clean ou Onion*

**P6. Qual o nível de experiência médio da equipe com arquitetura?**
- A) Júnior/Pleno, sem experiência com arquiteturas avançadas → *Hexagonal*
- B) Sênior, familiarizado com Clean Code e SOLID → *Clean*
- C) Sênior+, com experiência em DDD → *Onion*

**P7. Qual o prazo de vida esperado do projeto?**
- A) Curto prazo (< 1 ano, POC, MVP) → *Hexagonal*
- B) Médio prazo (1–3 anos) → *Clean ou Hexagonal*
- C) Longo prazo (> 3 anos, produto de core business) → *Clean ou Onion*

---

### Bloco 4 — Testabilidade e Qualidade

**P8. Qual importância você dá para testes automatizados?**
- A) Baixa: apenas testes de integração básicos → *qualquer*
- B) Média: testes unitários e de integração → *Hexagonal*
- C) Alta: cobertura elevada, TDD, testes de domínio isolados → *Clean ou Onion*

**P9. Você precisa trocar tecnologias facilmente (ex: trocar JPA por MongoDB)?**
- A) Não → *qualquer*
- B) Talvez → *Hexagonal*
- C) Sim, é um requisito → *Hexagonal (forte indicação)*

---

## 🤖 Prompt de IA para Decisão Automática

Cole o prompt abaixo em seu agente de IA preferido (Claude, GPT-4, etc.) para obter uma recomendação personalizada:

```
Você é um especialista em arquitetura de software Java e Spring Boot.
Vou responder um questionário e você deve me recomendar a melhor arquitetura
entre: Hexagonal (Ports & Adapters), Clean Architecture ou Onion Architecture.

Ao final, justifique sua escolha com base nas minhas respostas e me forneça:
1. A arquitetura recomendada
2. Justificativa baseada nas minhas respostas
3. Pontos de atenção específicos para meu contexto
4. Qual arquivo de instrução usar: HEXAGONAL_ARCH.md, CLEAN_ARCH.md ou ONION_ARCH.md

Aqui estão minhas respostas:

P1 - Complexidade do domínio: [sua resposta]
P2 - Especialista de domínio envolvido: [sua resposta]
P3 - Fontes de entrada: [sua resposta]
P4 - Integrações externas: [sua resposta]
P5 - Tamanho da equipe: [sua resposta]
P6 - Nível de experiência: [sua resposta]
P7 - Prazo de vida do projeto: [sua resposta]
P8 - Importância de testes: [sua resposta]
P9 - Necessidade de trocar tecnologias: [sua resposta]
```

---

## 📁 Estrutura dos Arquivos de Instrução

```
java-arch-ai-instructions/
├── README.md                  ← Este arquivo (guia de decisão)
├── HEXAGONAL_ARCH.md          ← Instrução para Hexagonal Architecture
├── CLEAN_ARCH.md              ← Instrução para Clean Architecture
└── ONION_ARCH.md              ← Instrução para Onion Architecture
```

---

## ⚡ Decisão Rápida

| Se seu projeto é... | Use |
|---|---|
| Microservice com múltiplos adapters (REST + Kafka + DB) | **HEXAGONAL_ARCH.md** |
| API simples com alguma lógica de negócio | **HEXAGONAL_ARCH.md** |
| Sistema enterprise com domínio complexo, equipe grande | **CLEAN_ARCH.md** |
| DDD aplicado, domínio rico, especialistas de negócio | **ONION_ARCH.md** |
| POC / MVP / curto prazo | **HEXAGONAL_ARCH.md** |
| Produto estratégico de longo prazo | **CLEAN_ARCH.md** ou **ONION_ARCH.md** |
