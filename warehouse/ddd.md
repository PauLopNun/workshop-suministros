# Warehouse — Pau (Core Domain)

Central node of the system. Manages inventory, minimum stock rules and replenishment requests.

## Modules

### Module: warehouse

```mermaid
classDiagram
    class Warehouse {
        <<aggregate root>>
        +WarehouseId id
        +String name
        +Location location
        +boolean isInfiniteStock
        +List~StockItem~ stock
        +List~StockRule~ stockRules
        +checkOwnStock(items) boolean
        +consumeStock(items)
        +receiveDelivery(items)
        +needsReplenishment() boolean
    }
    class StockItem {
        <<entity>>
        +String productId
        +int quantity
        +isEnough(needed) boolean
        +add(qty)
        +subtract(qty)
    }
    class StockRule {
        <<value object>>
        +String productId
        +int minQuantity
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    class WarehouseId {
        <<value object>>
        +String value
    }
    Warehouse --> WarehouseId
    Warehouse --> Location
    Warehouse "1" --> "*" StockItem
    Warehouse "1" --> "*" StockRule
```

### Module: replenishment

```mermaid
classDiagram
    class ReplenishmentRequest {
        <<aggregate root>>
        +ReplenishmentId id
        +String originWarehouseId
        +String destWarehouseId
        +List~RequestedItem~ items
        +ReplenishmentStatus status
        +assign()
        +complete()
    }
    class RequestedItem {
        <<value object>>
        +String productId
        +int quantity
    }
    class ReplenishmentStatus {
        <<value object>>
        PENDING
        IN_TRANSIT
        COMPLETED
    }
    class ReplenishmentId {
        <<value object>>
        +String value
    }
    ReplenishmentRequest --> ReplenishmentId
    ReplenishmentRequest --> ReplenishmentStatus
    ReplenishmentRequest "1" --> "*" RequestedItem
```

## Domain services

```mermaid
classDiagram
    class StockChecker {
        <<domain service>>
        +checkOwnStock(warehouse, items) boolean
    }
    class ReplenishmentPolicy {
        <<domain service>>
        +shouldReplenish(rule, currentQty) boolean
    }
    class SupplierWarehouseFinder {
        <<domain service>>
        +findSupplier(productId) String
    }
```

## Decision logic — production.materials.requested.v1

```mermaid
flowchart TD
    IN([production.materials.requested.v1 received]) --> A
    A{Own stock sufficient?}
    A -->|YES| B[Deducts stock\nPublishes warehouse.stock.changed.v1]
    A -->|NO| C[Finds supplier warehouse]
    C --> D{Supplier found?}
    D -->|YES| E[Creates ReplenishmentRequest\nPublishes replenishment.requested.v1\nPublishes dispatch.requested.v1]
    D -->|NO| F[Publishes production.order.blocked.v1]
    B --> Z([Production starts manufacturing])
    E --> G([Transport assigns a truck])
    F --> H([Reporting registers blocked order])
```

## Events published

| Event | Consumed by |
|---|---|
| replenishment.requested.v1 | Production, Reporting |
| warehouse.stock.changed.v1 | Production, Reporting |
| dispatch.requested.v1 | Transport |
| warehouse.registered.v1 | Time/Map |

## Package structure

```
warehouse-service/
├── warehouse/
│   ├── domain/
│   │   ├── Warehouse.java
│   │   ├── StockItem.java
│   │   ├── StockRule.java
│   │   ├── WarehouseId.java
│   │   ├── Location.java
│   │   └── service/
│   │       ├── StockChecker.java
│   │       ├── ReplenishmentPolicy.java
│   │       └── SupplierWarehouseFinder.java
│   ├── application/usecase/
│   │   ├── HandleMaterialsRequested.java
│   │   ├── HandleDeliveryCompleted.java
│   │   ├── HandleTimeAdvanced.java
│   │   └── RegisterWarehouse.java
│   └── infrastructure/
│       ├── rest/WarehouseController.java
│       ├── persistence/WarehouseJpaRepository.java
│       └── messaging/MaterialsRequestedListener.java
└── replenishment/
    ├── domain/
    │   ├── ReplenishmentRequest.java
    │   ├── ReplenishmentId.java
    │   ├── RequestedItem.java
    │   └── ReplenishmentStatus.java
    ├── application/usecase/CreateReplenishmentRequest.java
    └── infrastructure/messaging/ReplenishmentPublisher.java
```
