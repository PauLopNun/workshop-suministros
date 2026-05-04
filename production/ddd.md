# Production — Idoia

Manages factories, recipes, production orders and store orders.

## Core Domain Model

```mermaid
classDiagram
    %% =======================
    %% FACTORY AGGREGATE
    %% =======================
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
        +UUID id
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    
    Factory --> FactoryId
    Factory --> Location
    %% =======================
    %% PRODUCTION ORDER AGGREGATE
    %% =======================
    class ProductionOrder {
        <<aggregate root>>
        +OrderId id
        +FactoryId factoryId
        +RecipeId recipeId
        +WarehouseOrderId warehouseOrderId
        +OrderStatus status
        +int remainingDays
        +advance(days) DomainEvent
        +isCompleted() boolean
        +start()
    }
    class OrderId {
        <<value object>>
        +UUID id
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
    %% =======================
    %% RECIPE & INGREDIENT
    %% =======================
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
        <<entity>>
        +IngredientId id
        +String name
    }
    class IngredientId {
        <<value object>>
        +UUID id
    }
    class RecipeId {
        <<value object>>
        +UUID id
    }
    class MaterialRequest {
        <<entity>>
        +MaterialRequestId id
        +List~RecipeIngredients~ itemsRequest
    }
    class MaterialRequestId {
        <<value object>>
        +UUID id
    }
    
    Recipe --> RecipeId
    Recipe --> RecipeIngredient : contains
    RecipeIngredient --> IngredientId
    Ingredient --> IngredientId
    MaterialRequest --> MaterialRequestId
    MaterialRequest --> RecipeIngredient
    
    %% =======================
    %% WAREHOUSE ORDER AGGREGATE
    %% =======================
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
        +UUID id
    }
    class WarehouseOrderStatus {
        <<value object>>
        PENDING
        IN_PRODUCTION
        COMPLETED
    }
    
    WarehouseOrder --> WarehouseOrderStatus
    WarehouseOrder --> WarehouseOrderId

    %% =======================
    %% PRODUCT
    %% =======================
    class Product {
        <<entity>>
        +ProductId id
        +String name
        +String description
    }
    class ProductId {
        <<value object>>
        +UUID id
    }
    
    Product --> ProductId
    %% =======================
    %% CROSS-AGGREGATE RELATIONSHIPS
    %% =======================
    Factory ..> RecipeId : assigns
    ProductionOrder ..> FactoryId : placed at
    ProductionOrder ..> RecipeId : uses
    ProductionOrder ..> WarehouseOrderId : fulfills
    Recipe ..> ProductId : produces
    WarehouseOrder ..> ProductId : requests
```

## Events Schema

```mermaid
flowchart TD
    UC1([POST /factories]) --> A1[RegisterFactory]
    A1 --> A2[Creates and persists a Factory\nPublishes factory.registered.v1]
    UC2([POST /recipes]) --> B1[RegisterRecipe]
    B1 --> B2[Creates and persists a Recipe\nwith ingredients and output product\nPublished recipe.registered.v1]
    UC3([POST /warehouses/id/orders]) --> C1[PlaceWarehouseOrder]
    C1 --> C2[Creates ProductionOrder\nPublishes production.materials.requested.v1]
    UC4([time.advanced.v1 received]) --> D1[AdvanceProductionOrders]
    D1 --> D2[Decrements remainingDays for each IN_PROGRESS order]
    D2 --> D3{remainingDays == 0?}
    D3 -->|YES| D4[Publishes production.order.completed.v1\nPublishes dispatch.requested.v1]
    UC5([delivery.completed.v1 received]) --> E1[HandleMaterialsArrived]
    E1 --> E2[Publishes production.order.started.v1]
    UC6([replenishment.requested.v1 received]) --> F1[CheckIfProductionCanStart]
    UC7([GET /recipes]) --> G1[GetAllRecipes]
    G1 --> G2[Gets all recipes so warehouses can see if the factory can produce what they need]
```

## Use case

```mermaid
flowchart TD
    A0[Receives a order from the warehouse] --> A1[Search materials from warehouse]
    --> A2[Receive the materials from warehouse] --> A3[Crafts the products]
    --> A4[Send the products to the warehouse]
```

## Event Table

### Events Sent

| Event | Consumed by | Type |
|---|---|---|
| recipe.registered.v1 | Reporting | Factory |
| production.materials.requested.v1 | Warehouse | MaterialsRequested |
| production.order.completed.v1 | Warehouse, Reporting | WarehouseOrder |
| production.order.created.v1 | Reporting | ProductionOrder |
| production.order.started.v1 | Reporting | ProductionOrder |
| production.order.blocked.v1 | Reporting | ProductionOrder |

### Events Consumed

| Event | Received by | Type |
|---|---|---|
| time.advanced.v1 | Time |  |
| delivery.completed.v1 | Warehouse | boolean |
| replenishment.requested.v1 | Warehouse | ReplenishRequest |
