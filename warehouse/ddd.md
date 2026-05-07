# Warehouses — Bounded Context (Core Domain)
 
## Module: warehouse
 
```mermaid
classDiagram
    class Warehouse {
        <<aggregate root>>
        +WarehouseId id
        +String name
        +Location location
        +Map minStockRules
        +boolean isStockInfinite
        +checkOwnStock(items) boolean
        +consumeStock(items) void
        +receiveDelivery(items) void
        +needsReplenishment() boolean
        +dispatchItems(items) void
    }
    class StockItem {
        <<entity>>
        +ProductId productId
        +Quantity quantity
        +isEnough(needed) boolean
        +add(qty) void
        +subtract(qty) void
        +hasProduct(productId) boolean
    }
    class WarehouseId {
        <<value object>>
        +String value
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
        +String value
    }
    Warehouse "1" --> "1..*" StockItem
    Warehouse --> WarehouseId
    Warehouse --> Location
    StockItem --> ProductId
    StockItem --> Quantity
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
 
## Decision logic — production.materials.requested.v1
 
```mermaid
flowchart TD
    START([production.materials.requested.v1 received])
    Q1{Does the warehouse\nhave enough own stock?}
    YES1[Reserve stock\npublish warehouse.materials.reserved.v1]
    NO1[If no supplier can fulfill\npublish production.order.blocked.v1]
    Q2{Found another warehouse\nwith sufficient stock?}
    YES2[Publish shipment.requested.v1\nshipmentId + origin + destination + items -> Transport]
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
| shipment.requested.v1 | Transport |
| warehouse.registered.v1 | Time/Map |
