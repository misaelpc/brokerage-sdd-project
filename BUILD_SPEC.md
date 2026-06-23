# brokerage-core-api — Build Specification

> **Purpose:** Phased implementation guide for the simulated trading backend **brokerage-core-api**, built with **Quarkus**, **DDD + Hexagonal Architecture**, **>90% test coverage**, and **GitHub Actions CI**.
>
> **Sources:** `ddd_skill.md` *(mandatory for agents)*, `DesignTradingDDD.md`, `FUNCTIONAL_SPEC.md`
>
> **Target stack:** Java 25 · Quarkus 3.x · GraphQL · PostgreSQL · JUnit 5 · Mockito · Testcontainers

---

## Table of Contents

1. [Overview](#1-overview)
   - [1.0 Agent instructions (mandatory)](#10-agent-instructions-mandatory)
   - [1.5 Repository layout](#15-repository-layout)
2. [Phase 1 — Quarkus Project Creation](#2-phase-1--quarkus-project-creation)
3. [Phase 2 — DDD with Hexagonal Architecture](#3-phase-2--ddd-with-hexagonal-architecture)
4. [Phase 3 — Testing Plan (>90% Coverage)](#4-phase-3--testing-plan-90-coverage)
5. [Phase 4 — CI with GitHub Actions](#5-phase-4--ci-with-github-actions)
6. [Delivery Milestones](#6-delivery-milestones)
7. [Appendix A — GraphQL Contract](#appendix-a--graphql-contract)
8. [Appendix B — Seed Data](#appendix-b--seed-data)

---

## 1. Overview

### 1.0 Agent instructions (mandatory)

When implementing **any phase** of this build specification, the agent **must always follow** [`ddd_skill.md`](./ddd_skill.md). Read it at the start of each session and apply it to every code change, test, and refactor.

**Project location:** Scaffold and implement all application code under **`brokerage-core-api/`** (see [§1.5 Repository layout](#15-repository-layout)). Do not place Quarkus/Maven artifacts at the repository root.

`ddd_skill.md` is the authoritative source for:

| Area | What to follow |
|------|----------------|
| **DDD** | Ubiquitous language, small aggregates, identity references, invariants inside aggregate boundaries |
| **Layers** | Interior (`domain`, `application`) vs Exterior (`infra`) — no framework imports in domain |
| **Naming** | PascalCase classes, camelCase methods, package layout per DDD layers |
| **Code style** | Factory methods, `Optional`/Null Object (never `null` from domain), fail-fast guards |
| **Testing** | Given/When/Then, camelCase test names, required JUnit assertions, `@DisplayName` in ubiquitous language |
| **Security & CI** | Input validation, OWASP practices, branch-by-abstraction, feature toggles |

**Conflict resolution:** If this document and `ddd_skill.md` disagree on *coding standards or structure*, follow `ddd_skill.md`. If they disagree on *project scope, phases, or deliverables*, follow `BUILD_SPEC.md`. Document any intentional exception to `ddd_skill.md` rules (e.g., in a comment or ADR).

### 1.1 Product goal

**brokerage-core-api** is a simulated stock-trading backend inspired by an investment platform (GBM-style). Clients interact via **GraphQL over HTTP**. The system supports:

- Investor registration and profile management
- Financial instrument catalog and simulated market prices
- Buy/sell order lifecycle (market and limit orders)
- Portfolio positions and cash account (available + reserved balances)
- Activity feed for user-visible movements

Trading is **simulated** — no real market execution.

### 1.2 Architectural principles

| Principle | Requirement |
|-----------|---------------|
| DDD | Ubiquitous language, small aggregates, invariants inside aggregate boundaries |
| Hexagonal | Domain + Application are ports; Infrastructure + Presentation are adapters |
| Framework isolation | Domain layer has **zero** Quarkus/JPA/GraphQL dependencies |
| Identity references | Aggregates reference each other by ID only |
| Consistency | Strong inside an aggregate; eventual between aggregates via domain events |
| Null safety | Use `Optional` or Null Object Pattern — never return `null` from domain |
| Factory methods | Prefer static factories over public constructors on aggregates and value objects |

### 1.3 Bounded contexts

| Context | Core responsibility |
|---------|---------------------|
| **Investor Management** | Register investor, status (ACTIVE / SUSPENDED / CLOSED) |
| **Market Catalog** | Instruments (symbol, type, status) |
| **Market Data** | Simulated last price per instrument |
| **Trading** (Core Domain) | Order lifecycle: place, accept, reject, execute, cancel |
| **Portfolio** | Positions per investor |
| **Cash Account** | Available balance, reserved balance, deposits |
| **Activity Feed** | Denormalized user activity history |

### 1.4 Hexagonal mapping

```
                    ┌─────────────────────────────────────────┐
  Driving adapters  │           APPLICATION CORE              │  Driven adapters
  (Primary ports)   │  ┌─────────────┐  ┌─────────────────┐  │  (Secondary ports)
                    │  │ Application │  │     Domain      │  │
  GraphQL ─────────►│  │  Services   │──│  Aggregates,    │──┼──► Repository impl (JPA)
  (infra/controller)│  │  Commands   │  │  VOs, Events,   │  │
                    │  │  Queries    │  │  Domain Services│  │
                    │  └─────────────┘  │  Repo interfaces│──┼──► Event publisher
                    │                   └─────────────────┘  │
                    └─────────────────────────────────────────┘
```

| Hexagonal term | Project package | Responsibility |
|----------------|-----------------|----------------|
| **Domain** | `domain.*` | Pure business model — no framework imports |
| **Application (use cases)** | `application.*` | Orchestration, commands, DTOs |
| **Driving adapter** | `infra.controller.graphql` | GraphQL resolvers → application services |
| **Driven adapter — persistence** | `infra.persistence` | JPA/Panache repository implementations |
| **Driven adapter — events** | `infra.event` | Domain event publication (in-process initially) |
| **Cross-cutting** | `infra.config`, `infra.problem` | Seeding, error mapping |

### 1.5 Repository layout

The repository root holds **specification and agent guidance** only (`BUILD_SPEC.md`, `ddd_skill.md`, etc.). The runnable Quarkus application **must** be scaffolded in a dedicated subdirectory:

```text
<repository-root>/
├── BUILD_SPEC.md
├── ddd_skill.md
├── …other specs…
└── brokerage-core-api/          ← Quarkus Maven project (application root)
    ├── pom.xml
    ├── mvnw
    ├── compose.yaml
    ├── src/
    └── …
```

| Term | Path | Contents |
|------|------|----------|
| **Repository root** | `.` | Build specs, design docs, CI config (`.github/`) |
| **Application root** | `brokerage-core-api/` | All Quarkus/Maven source, config, Docker Compose, and tests |

**Do not** generate Quarkus project files (`pom.xml`, `src/`, `mvnw`, etc.) at the repository root. All paths in Phases 1–4 are relative to **`brokerage-core-api/`** unless explicitly noted as repository root (e.g. `.github/workflows/`).

---

## 2. Phase 1 — Quarkus Project Creation

### 2.1 Generate the project

From the **repository root**, create the Quarkus project inside **`brokerage-core-api/`** using the Quarkus CLI or [code.quarkus.io](https://code.quarkus.io) with these settings:

| Setting | Value |
|---------|-------|
| **Project directory** | `brokerage-core-api` |
| **Group** | `com.brokerage` |
| **Artifact** | `brokerage-core-api` |
| **Version** | `1.0.0-SNAPSHOT` |
| **Java version** | `25` |
| **Build tool** | Maven |
| **Package name** | `com.brokerage.core` |

**Extensions to add at creation:**

| Extension | Purpose |
|-----------|---------|
| `quarkus-smallrye-graphql` | GraphQL API |
| `quarkus-hibernate-orm-panache` | JPA persistence |
| `quarkus-jdbc-postgresql` | PostgreSQL driver |
| `quarkus-flyway` | Schema migrations |
| `quarkus-hibernate-validator` | Input validation |
| `quarkus-smallrye-health` | Liveness/readiness probes |
| `quarkus-micrometer-registry-prometheus` | Metrics (optional, recommended) |
| `quarkus-jacoco` | Code coverage reports |
| `quarkus-junit5` | Testing |
| `quarkus-junit5-mockito` | Mockito integration |
| `rest-assured` (test scope) | GraphQL HTTP tests |

**CLI equivalent** (run from repository root; creates `brokerage-core-api/`):

```bash
quarkus create app com.brokerage:brokerage-core-api:1.0.0-SNAPSHOT \
  --output=brokerage-core-api \
  --extension='smallrye-graphql,hibernate-orm-panache,jdbc-postgresql,flyway,hibernate-validator,smallrye-health,jacoco,junit5,junit5-mockito' \
  --java=25 \
  --no-code
```

### 2.2 Maven `pom.xml` requirements

Add or verify these properties and plugins:

```xml
<properties>
    <maven.compiler.release>25</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jacoco.minimum.coverage>0.90</jacoco.minimum.coverage>
    <surefire.argLine></surefire.argLine>
</properties>
```

**Required test dependencies:**

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>
```

**JaCoCo plugin** (enforce ≥90% on bundle — see Phase 3):

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>${jacoco.minimum.coverage}</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 2.3 Application configuration

**`src/main/resources/application.properties`:**

```properties
# Application
quarkus.application.name=brokerage-core-api
quarkus.http.port=8080

# GraphQL
quarkus.smallrye-graphql.root-path=/graphql
quarkus.smallrye-graphql.ui.root-path=/graphiql
quarkus.smallrye-graphql.ui.always-include=true

# PostgreSQL — Docker Compose (dev)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=brokerage
quarkus.datasource.password=brokerage
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/brokerage

# Hibernate — schema managed by Flyway, not auto-DDL
quarkus.hibernate-orm.database.generation=none
quarkus.hibernate-orm.log.sql=false
quarkus.hibernate-orm.physical-naming-strategy=org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy

# Flyway
quarkus.flyway.migrate-at-start=true
quarkus.flyway.baseline-on-migrate=true

# Health
quarkus.smallrye-health.root-path=/q/health

# Logging
quarkus.log.category."com.brokerage".level=INFO
```

**`src/main/resources/application-test.properties`:**

```properties
quarkus.datasource.db-kind=postgresql
# Testcontainers overrides JDBC URL at runtime via @QuarkusTestResource
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.smallrye-graphql.ui.always-include=false
```

**`src/test/resources/application.properties`:**

```properties
# Disable dev-only seeding in unit tests unless explicitly enabled
brokerage.seed.enabled=false
```

### 2.4 Docker Compose (local dev)

PostgreSQL runs in a container. The database is **created automatically** by the image (`POSTGRES_DB`); **Flyway** applies schema migrations on app startup — no manual `createdb` required.

**`brokerage-core-api/compose.yaml`** (application root):

```yaml
services:
  postgres:
    image: postgres:17
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: brokerage
      POSTGRES_USER: brokerage
      POSTGRES_PASSWORD: brokerage
    volumes:
      - brokerage_pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U brokerage -d brokerage"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  brokerage_pg_data:
```

| Setting | Value |
|---------|-------|
| Host | `localhost` |
| Port | `5432` |
| Database | `brokerage` |
| Username | `brokerage` |
| Password | `brokerage` |

**Credentials in `application.properties` must match `compose.yaml`.**

**Start the database:**

```bash
docker compose up -d
```

**Verify connectivity:**

```bash
docker compose exec postgres psql -U brokerage -d brokerage -c "SELECT 1;"
```

**Reset procedure:** Stop the container, remove the volume, and restart so Flyway re-applies migrations and the seeder runs on a fresh schema.

```bash
docker compose down -v
docker compose up -d
```

CI tests continue to use Testcontainers (Phase 3) with ephemeral PostgreSQL instances.

### 2.5 Flyway baseline migration

**`src/main/resources/db/migration/V1__initial_schema.sql`** — create tables aligned with aggregates (simplified; expand per aggregate in Phase 2):

- `investors`
- `instruments`
- `market_prices`
- `trading_orders`
- `portfolios` + `positions`
- `cash_accounts`
- `activity_items`

Use `DECIMAL(19,4)` for money, `VARCHAR` for symbols/emails, UUID or BIGINT for IDs (pick one strategy and use consistently).

### 2.6 Project skeleton directories

Create empty package structure (Phase 2 fills implementation):

```text
brokerage-core-api/src/main/java/com/brokerage/core/
├── application/
│   ├── command/
│   ├── data/
│   └── service/
├── domain/
│   ├── model/
│   │   ├── investor/
│   │   ├── instrument/
│   │   ├── market/
│   │   ├── order/
│   │   ├── portfolio/
│   │   ├── cash/
│   │   └── shared/
│   ├── event/
│   ├── repository/
│   └── service/
└── infra/
    ├── config/
    ├── controller/
    │   ├── graphql/
    │   ├── model/
    │   └── problem/
    ├── event/
    └── persistence/

src/main/resources/
├── application.properties
├── graphql/
│   └── schema.graphqls
└── db/migration/

src/test/java/com/brokerage/core/
├── domain/
├── application/
├── infra/
│   ├── persistence/
│   └── controller/
└── support/
    └── PostgresTestResource.java
```

### 2.7 `.gitignore` essentials

Ensure these are ignored:

```text
target/
.idea/
*.iml
.env
.DS_Store
```

### 2.8 Phase 1 exit criteria

- [ ] From `brokerage-core-api/`: `docker compose up -d` starts PostgreSQL; `./mvnw quarkus:dev` starts on port 8080 and Flyway applies migrations
- [ ] GraphiQL available at `/graphiql`
- [ ] Health check responds at `/q/health`
- [ ] Flyway migrations apply cleanly on empty database
- [ ] `./mvnw verify` runs (even if tests are placeholders initially)

---

## 3. Phase 2 — DDD with Hexagonal Architecture

### 3.1 Implementation order (recommended)

Build inside-out: **domain first**, then **application**, then **adapters**.

| Step | Deliverable | Depends on |
|------|-------------|------------|
| 2.1 | Shared value objects (`Money`, `Quantity`, `Symbol`, `EmailAddress`, `Currency`) | — |
| 2.2 | `InvestorAggregate` + events + repository port | 2.1 |
| 2.3 | `InstrumentAggregate` + events + repository port | 2.1 |
| 2.4 | `MarketPriceAggregate` + events + repository port | 2.1, 2.3 |
| 2.5 | `CashAccountAggregate` + events + repository port | 2.1, 2.2 |
| 2.6 | `PortfolioAggregate` + `Position` entity + repository port | 2.1, 2.2 |
| 2.7 | `TradingOrderAggregate` + events + repository port | 2.1 |
| 2.8 | Domain services: `OrderEligibilityService`, `OrderExecutionPolicy` | 2.2–2.7 |
| 2.9 | Application commands + services | 2.2–2.8 |
| 2.10 | JPA persistence adapters | 2.2–2.7 |
| 2.11 | Domain event handlers (in-process) | 2.9 |
| 2.12 | GraphQL schema + resolvers | 2.9 |
| 2.13 | Data seeder + demo seed | 2.10 |

### 3.2 Domain layer — aggregates and invariants

Implement exactly as specified in `DesignTradingDDD.md` §6–§8. Key rules:

#### InvestorAggregate

- States: `ACTIVE`, `SUSPENDED`, `CLOSED`
- `ensureCanTrade()` — only ACTIVE investors may place orders
- Closed investors cannot be reactivated

#### InstrumentAggregate

- Symbol normalized to uppercase
- `ensureTradable()` — only ACTIVE instruments accept orders

#### MarketPriceAggregate

- Price > 0; new `quotedAt` must not precede current timestamp

#### TradingOrderAggregate (Core Domain)

- Order sides: `BUY`, `SELL`; types: `MARKET`, `LIMIT`
- Status flow: `PENDING` → `ACCEPTED` → `EXECUTED` | `CANCELLED` | `REJECTED`
- LIMIT orders require `limitPrice`; MARKET orders must not have limit price
- Executed orders cannot cancel; cancelled orders cannot execute

#### PortfolioAggregate

- Positions keyed by `InstrumentId`; quantity never negative
- Updates only from executed orders

#### CashAccountAggregate

- `availableBalance` + `reservedBalance`; neither negative
- Buy orders reserve cash; execution debits reserved; cancellation releases reservation
- Sell execution credits available balance

### 3.3 Domain events and cross-aggregate consistency

| Event | Producers | Consumers (application handlers) |
|-------|-----------|----------------------------------|
| `InvestorRegisteredEvent` | Investor | Open cash account, open portfolio, activity feed |
| `TradingOrderPlacedEvent` | Trading | Reserve cash (buy), activity feed |
| `TradingOrderExecutedEvent` | Trading | Update portfolio, debit/credit cash, activity feed |
| `TradingOrderCancelledEvent` | Trading | Release cash reservation, activity feed |
| `MarketPriceChangedEvent` | Market Data | Trigger limit order evaluation (future hook) |

**Pattern:** Application service persists aggregate → publishes event → synchronous in-process handler updates other aggregates (acceptable for demo; document as eventual-consistency simulation).

### 3.4 Application layer — commands and services

**Commands** (`application.command`):

| Command | Service |
|---------|---------|
| `RegisterInvestorCommand` | `RegisterInvestorApplicationService` |
| `RegisterInstrumentCommand` | `RegisterInstrumentApplicationService` |
| `PublishMarketPriceCommand` | `PublishMarketPriceApplicationService` |
| `PlaceBuyOrderCommand` | `PlaceOrderApplicationService` |
| `PlaceSellOrderCommand` | `PlaceOrderApplicationService` |
| `CancelOrderCommand` | `CancelOrderApplicationService` |
| `DepositCashCommand` | `DepositCashApplicationService` |
| `ExecutePendingOrdersCommand` | `ExecuteOrderApplicationService` |

**Queries** (read side — can use application services or dedicated query services):

| Query | Service |
|-------|---------|
| `GetPortfolioSummaryQuery` | `PortfolioQueryApplicationService` |
| `GetInvestorActivityQuery` | `ActivityQueryApplicationService` |
| Read-only catalog/balance (v1 compat) | `CatalogQueryApplicationService`, `BalanceQueryApplicationService` |

**Application service rules:**

- No business rules that belong in aggregates
- Load aggregates via repository ports
- Invoke domain behavior; persist; publish events
- Return `application.data` DTOs — never expose JPA entities to GraphQL

### 3.5 Infrastructure — persistence adapters

Each repository port gets a JPA implementation in `infra.persistence`:

| Port | Adapter | Notes |
|------|---------|-------|
| `InvestorRepository` | `JpaInvestorRepository` | Map aggregate ↔ entity in adapter |
| `InstrumentRepository` | `JpaInstrumentRepository` | `findBySymbol(Symbol)` |
| `MarketPriceRepository` | `JpaMarketPriceRepository` | One row per instrument |
| `TradingOrderRepository` | `JpaTradingOrderRepository` | Filter by investor, status |
| `PortfolioRepository` | `JpaPortfolioRepository` | Positions as embeddable/collection |
| `CashAccountRepository` | `JpaCashAccountRepository` | One account per investor |

**Mapping rule:** JPA entities live **only** in `infra.persistence.entity`. Domain aggregates never carry JPA annotations.

### 3.6 Infrastructure — GraphQL driving adapter

Place resolvers under `infra.controller.graphql`:

| Resolver | Maps to |
|----------|---------|
| `InvestorGraphQLController` | Register investor, get investor |
| `InstrumentGraphQLController` | Catalog queries |
| `MarketPriceGraphQLController` | Price queries + publish mutation |
| `TradingOrderGraphQLController` | Place/cancel/execute orders |
| `PortfolioGraphQLController` | Portfolio + estimated value |
| `CashAccountGraphQLController` | Balance, deposit |
| `ActivityGraphQLController` | Activity feed |

**Error mapping** (`infra.controller.problem`):

| Domain / application exception | GraphQL classification | HTTP equivalent |
|-------------------------------|--------------------------|-----------------|
| `IllegalArgumentException` | `BAD_REQUEST` | 400 |
| `IllegalStateException` | `BAD_REQUEST` | 400 |
| `NoSuchElementException` | `NOT_FOUND` | 404 |
| Instrument not found | `NOT_FOUND` | 404 |
| Unhandled | `INTERNAL_ERROR` | 500 |

### 3.7 v1 read-only compatibility (FUNCTIONAL_SPEC)

Implement these queries **first** as a vertical slice before full trading mutations:

| Query | Behavior |
|-------|----------|
| `balance` | Demo investor cash (maps to `CashAccountAggregate.availableBalance`) |
| `catalog` | All active instruments with last price |
| `instrument(id)` | By ID — `NOT_FOUND` if missing |
| `instrumentBySymbol(symbol)` | By symbol — `NOT_FOUND` if missing |
| `portfolio` | Positions with `marketValue = price × quantity` |

This validates hexagonal wiring before order mutations.

### 3.8 Dependency rules (enforce in code review)

```
infra  →  application  →  domain
infra  →  domain (only via repository interfaces)

FORBIDDEN:
  domain  →  infra
  domain  →  application
  domain  →  quarkus.*
  domain  →  jakarta.persistence.*
```

Optional: add ArchUnit test (Phase 3) to enforce package dependencies.

### 3.9 Phase 2 exit criteria

- [ ] All six aggregates implemented with factory methods and domain events
- [ ] All repository ports have JPA adapters
- [ ] Full GraphQL schema (queries + mutations) operational
- [ ] Demo seed data loaded on empty DB (see Appendix B)
- [ ] End-to-end flow: register → deposit → buy → execute → portfolio updated
- [ ] Domain package has zero framework imports

---

## 4. Phase 3 — Testing Plan (>90% Coverage)

### 4.1 Coverage target and scope

| Metric | Target | Enforced by |
|--------|--------|-------------|
| **Line coverage (bundle)** | ≥ **90%** | JaCoCo `check` goal in `verify` phase |
| **Domain layer** | ≥ **95%** | Team standard (critical business logic) |
| **Application layer** | ≥ **90%** | JaCoCo per-package rule (optional) |
| **Infra adapters** | ≥ **85%** | Integration tests compensate |

**Include in coverage:** `com.brokerage.core.domain`, `com.brokerage.core.application`, `com.brokerage.core.infra`

**Exclude from coverage gates:**

- Generated code
- `infra.persistence.entity.*` (mapping boilerplate — cover via repository integration tests)
- Main application bootstrap class

```xml
<!-- JaCoCo exclusions example -->
<excludes>
    <exclude>**/infra/persistence/entity/**</exclude>
    <exclude>**/BrokerageCoreApiApplication.class</exclude>
</excludes>
```

### 4.2 Test pyramid

```
        ┌─────────────┐
        │  E2E GraphQL │  ~5%  — critical user journeys
        ├─────────────┤
        │ Integration  │  ~25% — repos, GraphQL, Testcontainers
        ├─────────────┤
        │  Application │  ~25% — service orchestration with mocked ports
        ├─────────────┤
        │   Domain     │  ~45% — aggregates, VOs, domain services
        └─────────────┘
```

### 4.3 Test conventions (from ddd_skill.md)

- **Naming:** camelCase method names, e.g. `givenActiveInvestor_whenPlaceBuyOrder_thenOrderIsAccepted`
- **Structure:** Given / When / Then in every test
- **DisplayName:** Ubiquitous language, e.g. `@DisplayName("Un inversionista activo puede colocar una orden de compra de mercado")`
- **Assertions:** Every `@Test` must contain at least one JUnit `assert*()` or `fail()` — Mockito `verify()` alone does not count
- **No duplication:** Do not re-test persistence inside application tests if repository integration tests already cover it

### 4.4 Domain unit tests (highest priority)

**Location:** `src/test/java/com/brokerage/core/domain/`

| Test class | Scenarios |
|------------|-----------|
| `MoneyTest` | Creation, plus/minus, multiply, currency mismatch throws |
| `QuantityTest` | Positive values, comparison |
| `SymbolTest` | Uppercase normalization, empty/invalid rejected |
| `EmailAddressTest` | Valid format, lowercase normalization |
| `InvestorAggregateTest` | Register, activate, suspend, close, `ensureCanTrade` guards |
| `InstrumentAggregateTest` | Register, activate/deactivate, `ensureTradable` |
| `MarketPriceAggregateTest` | Publish initial, update, stale timestamp rejected |
| `TradingOrderAggregateTest` | All order types, state transitions, invalid transitions throw |
| `PortfolioAggregateTest` | Increase/decrease position, insufficient position rejected |
| `CashAccountAggregateTest` | Deposit, reserve, release, debit, credit, negative balance rejected |
| `OrderEligibilityServiceTest` | Active investor, sufficient cash/position rules |
| `OrderExecutionPolicyTest` | MARKET/LIMIT buy/sell execution conditions |

**Example pattern:**

```java
@Test
@DisplayName("Una orden ejecutada no puede cancelarse")
void givenExecutedOrder_whenCancel_thenThrowsIllegalStateException() {
    // Given
    var order = TradingOrderAggregate.placeBuyMarketOrder(/* ... */);
    order.accept();
    order.execute(Money.of(new BigDecimal("3200.00"), Currency.getInstance("MXN")));

    // When / Then
    assertThrows(IllegalStateException.class, () -> order.cancel(CancellationReason.userRequested()));
}
```

### 4.5 Application service tests

**Location:** `src/test/java/com/brokerage/core/application/`

Use **mocked repository ports** and verify orchestration:

| Test class | Focus |
|------------|-------|
| `RegisterInvestorApplicationServiceTest` | Creates investor, publishes event |
| `PlaceOrderApplicationServiceTest` | Eligibility check, reservation, event published |
| `ExecuteOrderApplicationServiceTest` | Execution policy, portfolio + cash updates |
| `CancelOrderApplicationServiceTest` | Cancellation + cash release |
| `DepositCashApplicationServiceTest` | Balance increase |

Validate: correct port interactions, domain events published, transactional boundaries (no duplicate persistence tests).

### 4.6 Integration tests — persistence

**Location:** `src/test/java/com/brokerage/core/infra/persistence/`

**Infrastructure:** `@QuarkusTest` + Testcontainers PostgreSQL via `@QuarkusTestResource(PostgresTestResource.class)`

| Test class | Scenarios |
|------------|-----------|
| `JpaInvestorRepositoryIT` | Save and find by ID |
| `JpaInstrumentRepositoryIT` | Save, find by symbol |
| `JpaTradingOrderRepositoryIT` | Save, find by investor, filter by status |
| `JpaPortfolioRepositoryIT` | Positions persisted correctly |
| `JpaCashAccountRepositoryIT` | Balances persisted correctly |

### 4.7 Integration tests — GraphQL

**Location:** `src/test/java/com/brokerage/core/infra/controller/`

Use Rest Assured POST to `/graphql`:

| Test class | Scenarios |
|------------|-----------|
| `BalanceQueryIT` | Seeded demo account returns Andrea + 100000 |
| `CatalogQueryIT` | Returns all seeded instruments |
| `InstrumentQueryIT` | NOT_FOUND for unknown id/symbol |
| `PortfolioQueryIT` | Empty portfolio initially; populated after trade flow |
| `RegisterInvestorMutationIT` | Creates investor |
| `PlaceBuyOrderMutationIT` | Accepts order, reserves cash |
| `ExecuteOrdersMutationIT` | Executes and updates portfolio |
| `CancelOrderMutationIT` | Cancels and releases cash |

### 4.8 Architecture test (optional but recommended)

**`ArchitectureTest.java`** with ArchUnit:

```java
noClasses().that().resideInAPackage("..domain..")
    .should().dependOnClassesThat().resideInAnyPackage("..infra..", "jakarta..", "io.quarkus..")
```

### 4.9 Coverage ramp-up plan

| Sprint / Week | Focus | Expected coverage |
|---------------|-------|-------------------|
| W1 | Value objects + Investor + Instrument aggregates | ~40% |
| W2 | TradingOrder + CashAccount + Portfolio aggregates | ~65% |
| W3 | Domain services + application services | ~80% |
| W4 | Repository ITs + GraphQL ITs + JaCoCo gate enabled | **≥90%** |

**Enable JaCoCo gate only after W3** to avoid blocking early development; add `jacoco.skip=false` explicitly in CI from W4 onward.

### 4.10 Phase 3 exit criteria

- [ ] `./mvnw verify` passes with JaCoCo ≥90% line coverage
- [ ] Every aggregate invariant has at least one negative-path test
- [ ] All GraphQL queries from Appendix A have IT coverage
- [ ] Critical mutations (register, deposit, buy, execute, cancel) have IT coverage
- [ ] No `@Test` without JUnit assertions (static analysis or review)
- [ ] CI runs full test suite with Testcontainers

---

## 5. Phase 4 — CI with GitHub Actions

### 5.1 Workflow file

Create **`.github/workflows/ci.yml`**:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

defaults:
  run:
    working-directory: brokerage-core-api

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 25
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'
          cache: maven

      - name: Verify (compile, test, coverage)
        run: ./mvnw -B verify -Djacoco.skip=false

      - name: Upload JaCoCo report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: brokerage-core-api/target/site/jacoco/

  coverage-report:
    name: Coverage Summary
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write
    steps:
      - name: Download JaCoCo report
        uses: actions/download-artifact@v4
        with:
          name: jacoco-report
          path: jacoco/

      - name: Add coverage comment
        uses: mi-kas/karma-jasmine-coverage-action@v1
        if: false  # Replace with jacoco PR comment action of choice
        # Alternative: use github-action-jacoco@v1 or codecov
```

### 5.2 Recommended additional workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | push, PR | Build, test, JaCoCo gate |
| `pr-labeler.yml` (optional) | PR opened | Auto-label by changed paths |
| `dependency-review.yml` (optional) | PR | GitHub dependency vulnerability scan |

### 5.3 CI validation checklist

The CI pipeline **must fail** when any of these occur:

| Check | Mechanism |
|-------|-----------|
| Compilation errors | Maven `compile` |
| Unit/integration test failures | Maven `test` + `integration-test` |
| Coverage below 90% | JaCoCo `check` in `verify` |
| Flyway migration syntax errors | Quarkus startup in `@QuarkusTest` |
| ArchUnit violations (if enabled) | `ArchitectureTest` |

### 5.4 Branch protection (GitHub settings)

Configure on `main`:

- Require status check: **Build & Test**
- Require branches to be up to date before merging
- Require pull request reviews (≥1)

### 5.5 Local pre-push parity

Developers should run before pushing (from `brokerage-core-api/`):

```bash
cd brokerage-core-api
./mvnw verify -Djacoco.skip=false
```

Optional: add Maven Wrapper if not generated by Quarkus CLI.

### 5.6 Phase 4 exit criteria

- [ ] CI runs on every PR and push to `main`/`develop`
- [ ] PR cannot merge if tests fail or coverage < 90%
- [ ] JaCoCo HTML report uploaded as artifact
- [ ] Testcontainers works in GitHub Actions (Docker available on `ubuntu-latest` — no extra config needed)
- [ ] README documents CI badge and local dev steps

---

## 6. Delivery Milestones

| Milestone | Phases | Deliverable |
|-----------|--------|-------------|
| **M1 — Bootstrap** | Phase 1 | Runnable Quarkus app, PostgreSQL, Flyway, empty GraphQL |
| **M2 — Read model** | Phase 2 (partial) | v1 queries: balance, catalog, instrument, portfolio |
| **M3 — Domain core** | Phase 2 + 3 | All aggregates + domain tests ≥95% |
| **M4 — Trading flows** | Phase 2 + 3 | Full mutations + application tests |
| **M5 — Production-ready demo** | Phase 3 + 4 | ≥90% coverage, CI green, seed data, GraphiQL demo |

### Demo script (acceptance)

1. `cd brokerage-core-api && docker compose up -d` (PostgreSQL container; DB created automatically)
2. `./mvnw quarkus:dev` (Flyway applies schema migrations on startup)
3. Open GraphiQL → query `catalog` and `balance`
4. `registerInvestor` (or use seeded Andrea López)
5. `depositCash` 100,000 MXN
6. `placeBuyOrder` AAPL market order
7. `executePendingOrders`
8. Query `portfolio` — AAPL position visible
9. `placeSellOrder` + execute — cash credited

---

## Appendix A — GraphQL Contract

Full schema as defined in `DesignTradingDDD.md` §13. File: `brokerage-core-api/src/main/resources/graphql/schema.graphqls`.

**Queries:** `investor`, `instruments`, `instrument`, `marketPrice`, `portfolio`, `cashAccount`, `orders`, `activityFeed`, plus v1 aliases `balance`, `catalog`, `instrumentBySymbol`.

**Mutations:** `registerInvestor`, `depositCash`, `registerInstrument`, `publishMarketPrice`, `placeBuyOrder`, `placeSellOrder`, `cancelOrder`, `executePendingOrders`.

**Error contract:** `NOT_FOUND` for missing instruments/investors/orders; `BAD_REQUEST` for business rule violations.

---

## Appendix B — Seed Data

Aligned with `DesignTradingDDD.md` §19 and `FUNCTIONAL_SPEC.md` §8:

### Investor (demo)

| Field | Value |
|-------|-------|
| Name | Andrea López |
| Email | andrea.lopez@example.com |
| Status | ACTIVE |
| Initial deposit | 100,000.00 MXN |

### Instruments

| Symbol | Name | Type | Market | Price (MXN) |
|--------|------|------|--------|-------------|
| AAPL | Apple Inc. | STOCK | NASDAQ | 3,200.00 |
| TSLA | Tesla Inc. | STOCK | NASDAQ | 4,500.00 |
| NVDA | NVIDIA Corp. | STOCK | NASDAQ | 6,100.00 |
| MSFT | Microsoft Corp. | STOCK | NASDAQ | 7,200.00 |
| IVVPESO | ETF S&P 500 MXN | ETF | BMV | 95.00 |

### Seeding rules

```
IF investors table has rows THEN skip seeding
ELSE seed investor + cash account + empty portfolio + instruments + market prices
```

Implement in `infra.config.DataSeeder` triggered on startup (`@ApplicationScoped`, `@Observes StartupEvent`), gated by `brokerage.seed.enabled=true` (default in dev profile).

---

## Appendix C — Key business rules (traceability)

| # | Rule | Validated by |
|---|------|--------------|
| 1 | Only active investors can trade | `InvestorAggregateTest`, `OrderEligibilityServiceTest` |
| 2 | Only active instruments are tradable | `InstrumentAggregateTest` |
| 3 | Cannot buy without sufficient cash | `CashAccountAggregateTest`, `PlaceOrderApplicationServiceTest` |
| 4 | Cannot sell without sufficient position | `PortfolioAggregateTest`, `PlaceOrderApplicationServiceTest` |
| 5 | Executed orders cannot cancel | `TradingOrderAggregateTest` |
| 6 | Cancelled orders cannot execute | `TradingOrderAggregateTest` |
| 7 | LIMIT orders require limit price | `TradingOrderAggregateTest` |
| 8 | MARKET orders have no limit price | `TradingOrderAggregateTest` |
| 9 | Available balance never negative | `CashAccountAggregateTest` |
| 10 | Position quantity never negative | `PortfolioAggregateTest` |
| 11 | Market price > 0 | `MarketPriceAggregateTest` |
| 12 | Relevant changes emit domain events | Application service tests |

---

*Document version: 1.0 — brokerage-core-api build specification*
