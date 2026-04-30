# Prouction — Idoia

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
    class FactoryLocation {
        <<value object>>
        +FactoryId factoryId
        +String name
        +Location location
    }
    Factory --> FactoryId
    Factory --> Location
    Factory --> FactoryLocation
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
    ProductionOrder --> OrderId
    ProductionOrder --> OrderStatus
    ProductionOrder --> OrderLine
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
    Recipe --> RecipeIngredient : contains
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
    WarehouseOrder --> WarehouseOrderId
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
    UC1([POST /factories]) --> A1[RegisterFactory]
    A1 --> A2[Creates and persists a Factory]
    UC2([POST /recipes]) --> B1[RegisterRecipe]
    B1 --> B2[Creates and persists a Recipe\nwith ingredients and output product]
    UC3([POST /warehouses/id/orders]) --> C1[PlaceWarehouseOrder]
    C1 --> C2[Creates ProductionOrder\nPublishes production.materials.requested.v1]
    UC4([time.advanced.v1 received]) --> D1[AdvanceProductionOrders]
    D1 --> D2[Decrements remainingDays for each IN_PROGRESS order]
    D2 --> D3{remainingDays == 0?}
    D3 -->|YES| D4[Publishes production.order.completed.v1\nPublishes dispatch.requested.v1]
    UC5([delivery.completed.v1 received]) --> E1[HandleMaterialsArrived]
    E1 --> E2[Publishes production.order.started.v1]
    UC6([replenishment.requested.v1 received]) --> F1[CheckIfProductionCanStart]
```

## Events sent

| Event | Consumed by |
|---|---|
| factory.registered.v1 | Time/Map, Reporting |
| dispatch.requested.v1 | Transport |
| production.materials.requested.v1 | Warehouse |
| production.order.completed.v1 | Warehouse, Reporting |
| production.order.created.v1 | Reporting |
| production.order.started.v1 | Reporting |
| production.order.blocked.v1 | Reporting |

### Events consumed

| Event | Received by |
|---|---|
| time.advanced.v1 | Time |
| delivery.completed.v1 | Transport, Warehouse |
| repelnishment.requested.v1 | Warehouse |
