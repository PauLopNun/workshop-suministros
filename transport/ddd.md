# Transport - Sergi

Manages the truck fleet and deliveries between supply-chain locations.
It assigns trucks to shipment requests, advances trucks on simulation ticks, and publishes events for maps, reporting, and delivery receivers.

---

## Modules

### Module: truck

`Truck` is the aggregate root. It owns identity, current location, operational status, capacity, load, speed, and the list of assigned deliveries.

`Location`, `TruckId`, and `TruckStatus` are value objects/enums accessed through `Truck`. `OptimalTruckSelector` is a domain service that selects the closest available truck with enough remaining capacity.

```mermaid
classDiagram
    class Truck {
        <<aggregate root>>
        +TruckId truckId
        +String name
        +Location location
        +TruckStatus status
        +int capacity
        +int currentLoad
        +int speed
        +List~DeliveryId~ deliveryIds
        +remainingCapacity() int
        +canAccept(items) boolean
    }
    class TruckId {
        <<value object>>
        +UUID value
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    class TruckStatus {
        <<enum>>
        AVAILABLE
        IN_TRANSIT
        DELIVERED
    }
    Truck --> TruckId
    Truck --> Location
    Truck --> TruckStatus
    Truck --> DeliveryId
```

Current behavior uses `AVAILABLE` and `IN_TRANSIT`. `DELIVERED` exists in the enum and persistence model, but the current flow returns a truck directly to `AVAILABLE` after its last delivery is completed.

---

### Module: delivery

`Delivery` is the aggregate root for an assigned shipment. It links a `shipmentId` from the external request with the assigned `TruckId`, origin/destination, items, assignment day, and optional completion day.

`DeliveryItem` and `DeliveryId` are value objects. `Location` and `TruckId` are shared with the truck module.

```mermaid
classDiagram
    class Delivery {
        <<aggregate root>>
        +DeliveryId deliveryId
        +UUID shipmentId
        +TruckId truckId
        +Location origin
        +Location destination
        +List~DeliveryItem~ items
        +int assignedAt
        +Integer completedAt
        +isCompleted() boolean
        +isArrived(truckLocation) boolean
        +complete(completedAt) Delivery
    }
    class DeliveryItem {
        <<value object>>
        +String materialType
        +int quantity
    }
    class DeliveryId {
        <<value object>>
        +UUID value
    }
    Delivery --> DeliveryId
    Delivery --> TruckId
    Delivery --> Location
    Delivery "1" --> "*" DeliveryItem
```

---

## Domain services

```mermaid
classDiagram
    class OptimalTruckSelector {
        <<domain service>>
        +select(trucks, origin, requiredItems) Truck
    }
    class DistanceCalculator {
        <<domain service>>
        +calculate(from, to) int
        +isOnRoute(point, from, to) boolean
    }
```

`DistanceCalculator` uses Manhattan distance. Route checks assume movement on X first and then Y. `AdvanceTrucks` currently moves one grid unit per simulated day toward the first pending delivery for the truck.

---

## Use cases

### UC1 - `shipment.requested.v1` received -> Assign truck

```mermaid
flowchart TD
    UC1([shipment.requested.v1 received])
    UC1 --> A0[Map message to AssignTruckCommand]
    A0 --> A1[AssignTruck use case]
    A1 --> A2[Calculate requiredItems as sum of item quantities]
    A2 --> A3[OptimalTruckSelector.select closest AVAILABLE truck with capacity]
    A3 --> A4{Truck found?}
    A4 -->|NO| A5[Try IN_TRANSIT truck with remaining capacity]
    A5 --> A6{Truck found?}
    A6 -->|NO| A9[NoTruckAvailableException]
    A4 -->|YES| A7[Create Delivery and set truck IN_TRANSIT]
    A6 -->|YES| A8[Create Delivery and increase currentLoad]
    A7 --> A10[Publish truck.status.changed.v1 reason DISPATCHED]
    A8 --> A11[Publish truck.status.changed.v1 reason LOAD_UPDATED]
```

Implementation note: RabbitMQ binding for `shipment.requested.v1` exists, but `DispatchRequestedListener.java` is currently empty. The application use case is implemented; the messaging adapter still needs to map the event into `AssignTruckCommand`.

---

### UC2 - `time.advanced.v1` received -> Advance trucks

```mermaid
flowchart TD
    UC2([time.advanced.v1 received])
    UC2 --> B1{daysAdvanced > 0?}
    B1 -->|NO| B0[Ignore message]
    B1 -->|YES| B2[AdvanceTrucks use case]
    B2 --> B3[Find IN_TRANSIT trucks]
    B3 --> B4[For each simulated day, move one grid step toward first pending delivery]
    B4 --> B5[Publish truck.position.updated.v1]
    B5 --> B6{Arrived at delivery destination?}
    B6 -->|NO| B4
    B6 -->|YES| B7[Complete Delivery and publish delivery.completed.v1]
    B7 --> B8{More pending deliveries for truck?}
    B8 -->|YES| B9[Reduce currentLoad and publish truck.status.changed.v1 reason LOAD_UPDATED]
    B8 -->|NO| B10[Set truck AVAILABLE, currentLoad 0 and publish truck.status.changed.v1 reason RETURNED_TO_BASE]
```

`time.advanced.v1` payload currently contains `previousDayNumber`, `currentDayNumber`, and `daysAdvanced`. `currentDayNumber` is used as the completion/status timestamp.

---

### UC3 - Register truck through REST

```mermaid
flowchart TD
    UC3([POST /trucks])
    UC3 --> C1[RegisterTruck use case]
    C1 --> C2[Create Truck with status AVAILABLE and currentLoad 0]
    C2 --> C3[Persist truck]
    C3 --> C4[Publish truck.registered.v1]
    C4 --> C5[Publish truck.status.changed.v1 reason TRUCK_REGISTERED]
```

---

### UC4 - Initial state for Map UI

```mermaid
flowchart TD
    UC4([GET /trucks])
    UC4 --> D1[GetTrucks use case]
    D1 --> D2[Return all trucks with current location and status]
    D2 --> D3[Map UI renders initial state]
    D3 --> D4[Map UI continues with truck.position.updated.v1 and delivery.completed.v1]
```

Map UI is responsible for rendering trucks, not for registering them. Registration is done through `POST /trucks`.

---

## REST Endpoints

| Method | Path | Description | Consumer |
|---|---|---|---|
| `POST` | `/trucks` | Register a truck. Publishes `truck.registered.v1` and `truck.status.changed.v1` | Internal / Admin |
| `GET` | `/trucks` | Get all trucks with current location and status | Map UI |

---

## Events published

| Event | Exchange | Trigger | Consumed by |
|---|---|---|---|
| `truck.registered.v1` | `trucks.exchange` | Truck registration via `POST /trucks` | Reporting, Map UI |
| `truck.status.changed.v1` | `trucks.exchange` | Registration, dispatch, load update, all deliveries completed | Reporting |
| `truck.position.updated.v1` | `trucks.exchange` | Each simulated day while moving | Map UI, Reporting |
| `delivery.completed.v1` | `shipments.exchange` | Truck arrives at a delivery destination | Warehouses, Reporting, Map UI |

---

## Events consumed

| Event | Exchange | Queue | Use case triggered |
|---|---|---|---|
| `shipment.requested.v1` | `shipments.exchange` | `trucks.shipment.requested` | AssignTruck |
| `time.advanced.v1` | `simulation.exchange` | `trucks.time.advanced` | AdvanceTrucks |

---

## Persistence

Transport persists trucks and deliveries with PostgreSQL and Liquibase.

| Aggregate | Repository port | Infrastructure adapter |
|---|---|---|
| `Truck` | `TruckRepository` | `TruckRepositoryAdapter` + `TruckJpaRepository` |
| `Delivery` | `DeliveryRepository` | `DeliveryRepositoryAdapter` + `DeliveryJpaRepository` |
