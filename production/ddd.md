# Factories — Idoia

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
        +WarehouseId warehouseId
        +List~RecipeId~ assignedRecipes
        +canProduce(recipeId) boolean
    }
    class FactoryId {
        <<value object>>
        +String id
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
        +FactoryId factoryId
        +RecipeId recipeId
        +WarehouseOrderId warehouseOrderId
        +OrderStatus status
        +int remainingDays
        +List~OrderLine~ lines
        +advance(days) DomainEvent
        +isCompleted() boolean
        +start()
    }
    class OrderId {
        <<value object>>
        +String id
    }
    class OrderLine {
        <<entity>>
        +ProductId productId
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
        +String id
    }
    ProductionOrder --> OrderId
    ProductionOrder --> OrderStatus
    ProductionOrder "1" --> "*" OrderLine
    ProductionOrder --> OrderId
```

### Module: recipe

```mermaid
classDiagram
    class Recipe {
        <<aggregate root>>
        +RecipeId id
        +String name
        +List~RecipeIngredient~ inputs
        +ProductId outputProductId
        +int buildTimeInDays
    }
    class RecipeIngredient {
        <<value object>>
        +IngredientId ingredientId
        +int quantity
    }
    class Ingredient {
        <<aggregate root>>
        +IngredientId id
        +String name
    }
    class IngredientId {
        <<value object>>
        +String id
    }
    class RecipeId {
        <<value object>>
        +String id
    }
    Recipe --> RecipeId
    Recipe "1" --> "*" RecipeIngredient : contains
    RecipeIngredient --> IngredientId
    Ingredient --> IngredientId
```

### Module: warehouse-order

```mermaid
classDiagram
    class WarehouseOrder {
        <<aggregate root>>
        +WarehouseOrderId id
        +WarehouseId warehouseId
        +ProductId productId
        +int quantity
        +WarehouseOrderStatus status
        +complete()
        +cancel(reason)
    }
    class WarehouseOrderId {
        <<value object>>
        +String id
    }
    class WarehouseOrderStatus {
        <<value object>>
        PENDING
        IN_PRODUCTION
        SHIPPED
        COMPLETED
    }
    WarehouseOrder --> WarehouseOrderStatus
```

### Module: product
```mermaid
classDiagram
    class Product {
        <<aggregate root>>
        +ProductId id
        +String name
        +String description
        +double unitWeight
    }
    class ProductId {
        <<value object>>
        +String id
    }
    Product --> ProductId
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

