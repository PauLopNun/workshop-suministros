# Warehouse — Pau (Core Domain)

Central node of the system. Manages inventory, minimum stock rules and replenishment requests.

## Modules

### Module: warehouse

```mermaid
flowchart TD
    subgraph API["Layer: API REST"]
        WC["**WarehouseController**
        POST /warehouses
        GET /{id}/stock
        GET /warehouses"]
    end
 
    subgraph APP["Layer: Application (Use Cases)"]
        RW["**RegisterWarehouse**
        Registers a new warehouse."]
 
        GWS["**GetWarehouseStock**
        Queries current stock."]
 
        RD["**ReceiveDelivery**
        Handles DeliveryCompleted event.
        Registers incoming stock from a delivery
        and updates warehouse quantities."]
 
        DS["**DispatchShipment**
        Handles TruckDeparture event.
        Reduces stock on truck departure
        and records the shipment dispatch."]
    end
 
    subgraph DOM["Layer: Domain"]
        subgraph AGG["Aggregate Root"]
            WH["**Warehouse**
            ────────────────────────────
            id: WarehouseId
            name: String
            location: Location
            minStockRules: Map
            isStockInfinite: boolean
            ────────────────────────────
            + checkOwnStock(items): boolean
            + consumeStock(items): void
            + receiveDelivery(items): void
            + needsReplenishment(): boolean
            + dispatchItems(items): void"]
        end
 
        SI["**StockItem** — entity, internal to Warehouse
        ─────────────────────────────────
        productId: ProductId
        quantity: Quantity
        ─────────────────────────────────
        + isEnough(needed): boolean
        + add(qty): void
        + subtract(qty): void
        + hasProduct(productId): boolean"]
    end
 
    subgraph VO["Layer: Value Objects"]
        VOX["WarehouseId · Location (x, y) · Quantity (value > 0) · ProductId · Type ENUM"]
    end
 
    WC -->|"calls"| RW
    WC -->|"calls"| GWS
    WC -->|"calls"| RD
    WC -->|"calls"| DS
 
    RW  -->|"orchestrates"| WH
    GWS -->|"orchestrates"| WH
    RD  -->|"orchestrates"| WH
    DS  -->|"orchestrates"| WH
 
    WH  -->|"contains 1..*"| SI
    WH  -.->|"depends on"| VOX
    SI  -.->|"depends on"| VOX
 
    classDef apiStyle fill:#F1EFE8,stroke:#B4B2A9,color:#333
    classDef appStyle fill:#F1EFE8,stroke:#B4B2A9,color:#333
    classDef aggStyle fill:#e1f5ee,stroke:#1d9e75,color:#085041
    classDef voStyle  fill:#f5f5f5,stroke:#999,color:#444
 
    class WC apiStyle
    class RW,GWS,RD,DS appStyle
    class WH,SI aggStyle
    class VOX voStyle
```
 
---
 
## Module: Replenishment
 
```mermaid
flowchart TD
    subgraph APP["Layer: Application (Use Cases)"]
        HTT["**HandleTimeTick**
        Triggers auto-replenishment check
        on every time tick event."]
    end
 
    subgraph DOM["Layer: Domain"]
        subgraph AGG["Aggregate Root"]
            WH["**Warehouse**
            ────────────────────────────
            + needsReplenishment(): boolean
            + dispatchItems(items): void"]
        end
 
        subgraph SVC["Domain Services"]
            RP["**ReplenishmentPolicy**
            shouldReplenish() → boolean"]
 
            SWF["**SupplierWarehouseFinder**
            findSupplier(factoryRecipe) → WarehouseId?
            — finds supplier by factory recipe"]
        end
    end
 
    subgraph EVT["Events Published"]
        E5["**ReplenishmentRequested**
        { warehouseId, factoryRecipe, items[] }
        → Supplier"]
    end
 
    subgraph VO["Layer: Value Objects"]
        VOX["WarehouseId · Quantity (value > 0) · ProductId"]
    end
 
    HTT -->|"orchestrates"| WH
    WH  -.->|"uses"| RP
    WH  -.->|"uses"| SWF
    SWF -->|"publishes"| E5
    WH  -.->|"depends on"| VOX
 
    classDef appStyle fill:#F1EFE8,stroke:#B4B2A9,color:#333
    classDef aggStyle fill:#e1f5ee,stroke:#1d9e75,color:#085041
    classDef svcStyle fill:#f5f5f5,stroke:#999,color:#444
    classDef evtStyle fill:#e1f5ee,stroke:#1d9e75,color:#085041
    classDef voStyle  fill:#f5f5f5,stroke:#999,color:#444
 
    class HTT appStyle
    class WH aggStyle
    class RP,SWF svcStyle
    class E5 evtStyle
    class VOX voStyle
```
 
---
 
## Module: Services
 
```mermaid
flowchart TD
    subgraph SVC["Domain Services"]
        SC["**StockChecker**
        checkOwnStock() → boolean
        Verifies whether the warehouse
        holds enough stock for the request."]
 
        RP["**ReplenishmentPolicy**
        shouldReplenish() → boolean
        Evaluates min stock rules to decide
        if replenishment is needed."]
 
        SWF["**SupplierWarehouseFinder**
        findSupplier(factoryRecipe) → WarehouseId?
        Searches for a warehouse that can
        supply based on the factory recipe."]
    end
 
    subgraph AGG["Aggregate Root — Warehouse (consumer)"]
        WH["**Warehouse**
        + checkOwnStock(items): boolean
        + needsReplenishment(): boolean"]
    end
 
    WH -.->|"uses"| SC
    WH -.->|"uses"| RP
    WH -.->|"uses"| SWF
 
    classDef aggStyle fill:#e1f5ee,stroke:#1d9e75,color:#085041
    classDef svcStyle fill:#f5f5f5,stroke:#999,color:#444
 
    class WH aggStyle
    class SC,RP,SWF svcStyle
```
 
---
 
## Decision Logic — `production.materials.requested.v1`
 
```mermaid
flowchart TD
    START(["MaterialsNeeded event received"])
 
    Q1{"1. Does the warehouse
    have enough own stock?"}
 
    YES1["Publish **StockAvailable**
    { orderId, warehouseId }
    → Factories"]
 
    NO1["Publish **StockUnavailable**
    { orderId } → Reports
    Order blocked — search other warehouses"]
 
    Q2{"2. Found another warehouse
    with sufficient stock?"}
 
    YES2["Publish **ShipmentRequested**
    { shipmentId, originId, destId, items[], priority }
    → Trucks"]
 
    NO2["Fatal ERROR - No warehouse is producing stock,
    order canceled"]
 
    START --> Q1
    Q1 -->|"YES"| YES1
    Q1 -->|"NO"| NO1
    NO1 --> Q2
    Q2 -->|"YES"| YES2
    Q2 -->|"NO"| NO2
 
    classDef decision  fill:#fff9c4,stroke:#f9a825,color:#633806
    classDef published fill:#e1f5ee,stroke:#1d9e75,color:#085041
    classDef start     fill:#e8f0fe,stroke:#4a90d9,color:#1a3a5c
 
    class Q1,Q2 decision
    class YES1,YES2,NO1,REP published
    class START start
```



## Events published

| Event | Consumed by |
|---|---|
| replenishment.requested.v1 | Production, Reporting |
| warehouse.stock.changed.v1 | Production, Reporting |
| dispatch.requested.v1 | Transport |
| warehouse.registered.v1 | Time/Map |

