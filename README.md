# Supply Chain Simulator

Distributed system that simulates a supply chain between factories, warehouses and stores.

## Teams

| Service | Team member | Role |
|---|---|---|
| Time Passage + Map | Rubén | Open Host Service — upstream of all |
| Factories + Recipes | Idoia | Customer/Supplier — downstream of Time |
| Warehouses | Pau | Core Domain — central node |
| Trucks | Sergi | Conformist — downstream of Almacenes |
| Reports | Pedro | Anticorruption Layer — downstream of all |

## Context Map

![Context Map](context-map.png)

## Main flow — UC-05: Store orders 3 tables

```mermaid
flowchart TD
    S1([Store places order\nPOST /stores/id/orders]) --> S2
    S2[Production receives order\nand finds the recipe] --> S3
    S3{Are there tables\nin Warehouse?}
    S3 -->|YES| SC
    S3 -->|NO| S4
    SC[Truck delivers tables\ndirectly to Store] --> END
    S4{Are there materials\nin Warehouse?}
    S4 -->|YES| S5A
    S4 -->|NO| S5B
    S5A[Warehouse deducts materials\nStockAvailable to Production] --> S6
    S5B[Warehouse finds\nsupplier warehouse] --> S5C
    S5B -->|no supplier| BLK
    S5C[tick: Truck brings materials\nto factory warehouse] --> S6
    BLK([BLOCKED\nStockUnavailable to Reporting])
    S6[Production manufactures\n3 tables\ntakes N days per recipe] --> S7
    S7[tick: Tables ready\nTruck delivers to Store] --> END
    END([tick: Store receives tables\nOrder COMPLETED])

    style SC fill:#e8f5e9,stroke:#388e3c
    style END fill:#e8f5e9,stroke:#388e3c
    style BLK fill:#ffebee,stroke:#c62828
    style S3 fill:#fff9c4,stroke:#f9a825
    style S4 fill:#fff9c4,stroke:#f9a825
```

> tick = user clicks Advance Day

## Dependency rule (applies to all services)

```mermaid
graph LR
    I[infrastructure] --> A[application]
    A --> D[domain]
    style D fill:#e1f5ee,stroke:#1d9e75
    style A fill:#faeeda,stroke:#d6a427
    style I fill:#e6f1fb,stroke:#185fa5
```

`domain` never depends on JPA, RabbitMQ or any external framework.

## Naming conventions

### Branches

Format: `type/short-description`

Valid types: `feature` `fix` `chore` `docs` `test` `refactor`

Examples:

    feature/truck-assignment
    fix/stock-checker-null-pointer
    chore/setup-rabbitmq-config

### Commits

Format: `type: short description`

Valid types: `feat` `fix` `chore` `docs` `test` `refactor`

Examples:

    feat: add optimal truck selector
    fix: handle null stock when warehouse is empty
    chore: add liquibase migration for warehouse table

> Both conventions are enforced automatically by the CI pipeline on every push and pull request.
> 
## Tech stack

- Java 21 + Spring Boot 3
- PostgreSQL + Liquibase
- RabbitMQ (topic exchanges)
- SpringDoc OpenAPI
- JUnit 5 + AssertJ + Awaitility + Testcontainers
- Lombok
- Docker Compose
- GitHub Actions

---

## Definition of Ready

*A task is ready to be developed when:*

- [ ] The description is clear and the team understands it without needing to ask
- [ ] The messaging contracts it depends on are agreed and documented
- [ ] Dependencies with other teams are identified and unblocked
- [ ] Acceptance criteria are defined
- [ ] The team has estimated it

---

## Definition of Done

*A task is done when:*

- [ ] The code is on a branch following the correct format (`feature/`, `fix/`, etc.)
- [ ] Commits follow the format `type: short description`
- [ ] The PR is open, reviewed and approved by at least one team member
- [ ] The domain has no dependencies on JPA, RabbitMQ or any external framework
- [ ] Domain unit tests pass
- [ ] Integration tests with Testcontainers pass (RabbitMQ + PostgreSQL)
- [ ] A Liquibase migration is included if there are database changes
- [ ] Any published or consumed contract is documented in `contracts-doc.md`
- [ ] The GitHub Actions pipeline is green
