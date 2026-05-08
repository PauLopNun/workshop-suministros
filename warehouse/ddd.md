# Warehouses — Bounded Context (Core Domain)

## Module: warehouse

```mermaid
classDiagram
    class Warehouse {
        <<aggregate root>>
        +WarehouseId id
        +String name
        +Type warehouseType 
        +Location location
        +FactoryId id
        +checkOwnStock(items) boolean
        +consumeStock(items) List<stockItem>
        +receiveDelivery(items) void
        +dispatchItems(items) void
    }
    class StockItem {
        <<entity>>
        +StockItemID stockItemId
        +ProductId productId
        +Quantity quantity
        +WarehouseId warehouseId
        +Quantity minimumQuantityRule
        +isEnough(needed) boolean
        +add(qty) void
        +subtract(qty) void
        +hasProduct(productId) boolean
    }
    class WarehouseId {
        <<value object>>
        +UUID value
    }
    class Location {
        <<value object>>
        +Number x
        +Number y
    }
    class Quantity {
        <<value object>>
        +Number value
    }
    class ProductId {
        <<value object>>
        +UUID value
    }
    class StockItemId{
        <<value object>>
        +UUID id
    }
    class FactoryId {
        <<value object>>
        +UUID value
    }
    class Type {
        <<enumeration>>
        PRODUCTION
        FACTORY
        CLIENT
    }
    Warehouse "1" --> "1..*" StockItem
    Warehouse --> WarehouseId
    Warehouse --> Location
    Warehouse --> FactoryId
    Warehouse --> Type
    StockItem --> ProductId
    StockItem --> Quantity
    StockItem --> StockItemId
```

## Module: replenishment

```mermaid
classDiagram
    class Warehouse {
        <<aggregate root>>
        +needsReplenishment() boolean
        +dispatchItems(items) void
    }
    class ReplenishmentPolicy {
        <<domain service>>
        +shouldReplenish() boolean
    }
    class SupplierWarehouseFinder {
        <<domain service>>
        +findSupplier(factoryRecipe) WarehouseId
    }
    class HandleTimeTick {
        <<use case>>
        +handle(event) void
    }
    class ReplenishmentRequested {
        <<domain event>>
        +WarehouseId warehouseId
        +String factoryRecipe
        +Array items
    }
    HandleTimeTick --> Warehouse
    Warehouse ..> ReplenishmentPolicy
    Warehouse ..> SupplierWarehouseFinder
    SupplierWarehouseFinder --> ReplenishmentRequested
```

## Module: services

```mermaid
classDiagram
    class Warehouse {
        <<aggregate root>>
        +checkOwnStock(items) boolean
        +needsReplenishment() boolean
    }
    class StockChecker {
        <<domain service>>
        +checkOwnStock() boolean
    }
    class ReplenishmentPolicy {
        <<domain service>>
        +shouldReplenish() boolean
    }
    class SupplierWarehouseFinder {
        <<domain service>>
        +findSupplier(factoryRecipe) WarehouseId
    }
    Warehouse ..> StockChecker
    Warehouse ..> ReplenishmentPolicy
    Warehouse ..> SupplierWarehouseFinder
```

## Use cases (Application Layer)

```mermaid
classDiagram
    class RegisterWarehouse {
        <<use case>>
        +execute(command) void
    }
    class GetWarehouseStock {
        <<use case>>
        +execute(warehouseId) StockItem[]
    }
    class ReceiveDelivery {
        <<use case>>
        +handle(DeliveryCompleted) void
    }
    class DispatchShipment {
        <<use case>>
        +handle(TruckDeparture) void
    }
    class Warehouse {
        <<aggregate root>>
    }
    RegisterWarehouse --> Warehouse
    GetWarehouseStock --> Warehouse
    ReceiveDelivery --> Warehouse
    DispatchShipment --> Warehouse
```

## REST

```mermaid
flowchart LR
    P1([POST /warehouses]) --> UC1[RegisterWarehouse]
    UC1 --> R1["Creates and persists a Warehouse\nPublishes warehouse.registered.v1"]

    P2([GET /warehouses]) --> UC2[GetAllWarehouses]
    UC2 --> R2[Gets all Warehouses]

    P3(["GET /warehouses/{warehouseId}"]) --> UC3[GetWarehouseByWarehouseId]
    UC3 --> R3[Gets a Warehouse by its id]
```

## Decision logic — production.materials.requested.v1

```mermaid
flowchart TD
    START([MaterialsNeeded event received])
    Q1{Does the warehouse\nhave enough own stock?}
    YES1[Publish StockAvailable\norderId + warehouseId → Factories]
    NO1[Publish StockUnavailable\norderId → Reports\nOrder blocked — search other warehouses]
    Q2{Found another warehouse\nwith sufficient stock?}
    YES2[Publish ShipmentRequested\nshipmentId + originId + destId + items + priority → Trucks]
    NO2[Fatal error\nNo warehouse producing stock\nOrder canceled]
    START --> Q1
    Q1 -->|YES| YES1
    Q1 -->|NO| NO1
    NO1 --> Q2
    Q2 -->|YES| YES2
    Q2 -->|NO| NO2
```

### Events published

| Event | Consumed by |
|---|---|
| replenishment.requested.v1 | Production, Reporting |
| warehouse.stock.changed.v1 | Production, Reporting |
| dispatch.requested.v1 | Transport |
| warehouse.registered.v1 | Time/Map |
