# Production — Idoia

Manages factories, recipes, production orders and store orders.

## Modules

### Module: factory

```mermaid
classDiagram
    class Factory {
        <<aggregate root>>
        +FactoryId id
        +String name
        +Location location
        +String warehouseId
        +List~String~ assignedRecipes
        +canProduce(recipeId) boolean
    }
    class FactoryId {
        <<value object>>
        +String value
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    Factory --> FactoryId
    Factory --> Location
```

### Module: production-order

```mermaid
classDiagram
    class ProductionOrder {
        <<aggregate root>>
        +OrderId id
        +String factoryId
        +String recipeId
        +String storeOrderId
        +OrderStatus status
        +int remainingDays
        +List~OrderLine~ lines
        +advance(days) DomainEvent
        +isCompleted() boolean
        +start()
    }
    class OrderLine {
        <<entity>>
        +String productId
        +int quantity
        +boolean consumed
    }
    class OrderStatus {
        <<value object>>
        PENDING
        IN_PROGRESS
        COMPLETED
        BLOCKED
    }
    class OrderId {
        <<value object>>
        +String value
    }
    ProductionOrder --> OrderId
    ProductionOrder --> OrderStatus
    ProductionOrder "1" --> "*" OrderLine
```

### Module: recipe

```mermaid
classDiagram
    class Recipe {
        <<aggregate root>>
        +RecipeId id
        +String name
        +List~Ingredient~ inputs
        +String outputProductId
        +int buildTimeInDays
    }
    class Ingredient {
        <<value object>>
        +String productId
        +int quantity
    }
    class RecipeId {
        <<value object>>
        +String value
    }
    Recipe --> RecipeId
    Recipe "1" --> "*" Ingredient
```

### Module: store-order

```mermaid
classDiagram
    class StoreOrder {
        <<aggregate root>>
        +StoreOrderId id
        +String storeId
        +String productId
        +int quantity
        +StoreOrderStatus status
        +complete()
        +cancel(reason)
    }
    class Store {
        <<aggregate root>>
        +StoreId id
        +String name
        +Location location
        +String warehouseId
    }
    class StoreOrderStatus {
        <<value object>>
        PENDING
        IN_PRODUCTION
        SHIPPED
        COMPLETED
    }
    StoreOrder --> StoreOrderStatus
```

## Use cases

```mermaid
flowchart TD
    UC1([POST /stores/id/orders]) --> A1[PlaceStoreOrder]
    A1 --> A2[Creates ProductionOrder\nPublishes production.materials.requested.v1]

    UC2([time.advanced.v1 received]) --> B1[AdvanceProductionOrders]
    B1 --> B2[Decrements remainingDays for each IN_PROGRESS order]
    B2 --> B3{remainingDays == 0?}
    B3 -->|YES| B4[Publishes production.order.completed.v1\nPublishes dispatch.requested.v1]

    UC3([delivery.completed.v1 received]) --> C1[HandleMaterialsArrived]
    C1 --> C2[Publishes production.order.started.v1]

    UC4([replenishment.requested.v1 received]) --> D1[CheckIfProductionCanStart]
```

## Events published

| Event | Consumed by |
|---|---|
| production.order.created.v1 | Reporting |
| production.order.started.v1 | Reporting |
| production.order.blocked.v1 | Reporting |
| production.order.completed.v1 | Warehouse, Reporting |
| production.materials.requested.v1 | Warehouse |
| dispatch.requested.v1 | Transport |
| factory.registered.v1 | Time/Map |

## Package structure

```
production-service/
├── factory/
│   ├── domain/
│   │   ├── Factory.java
│   │   ├── FactoryId.java
│   │   └── Location.java
│   └── ...
├── production-order/
│   ├── domain/
│   │   ├── ProductionOrder.java
│   │   ├── OrderLine.java
│   │   ├── OrderId.java
│   │   └── OrderStatus.java
│   └── ...
├── recipe/
│   ├── domain/
│   │   ├── Recipe.java
│   │   ├── RecipeId.java
│   │   └── Ingredient.java
│   └── ...
└── store-order/
    ├── domain/
    │   ├── StoreOrder.java
    │   ├── Store.java
    │   └── StoreOrderStatus.java
    └── ...
```
