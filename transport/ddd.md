# Transport — Sergi

Manages the truck fleet and deliveries between warehouses.
Moves trucks on each time tick and notifies when a delivery is completed.

---

## Modules

### Module: truck

Each truck is the aggregate root of this module. `Location` and `TruckStatus` are value objects accessed only through `Truck`. `OptimalTruckSelector` is a domain service that crosses both modules to find the best truck given an origin location.

```mermaid
classDiagram
    class Truck {
        <<aggregate root>>
        +TruckId id
        +String name
        +Location location
        +TruckStatus status
        +int speed
        +int capacity
        +int currentLoad
        +List~DeliveryId~ deliveryIds
        +assignDelivery(deliveryId, itemCount) DomainEvent
        +completeDelivery(deliveryId) DomainEvent
        +hasCapacityFor(itemCount) boolean
        +isAvailable() boolean
        +isInTransit() boolean
    }
    class TruckId {
        <<value object>>
        +String value
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    class TruckStatus {
        <<value object>>
        AVAILABLE
        IN_TRANSIT
        DELIVERED
    }
    Truck --> TruckId
    Truck --> Location
    Truck --> TruckStatus
    Truck --> DeliveryId
```

---

### Module: delivery

`Delivery` is the aggregate root. `DeliveryItem` and `DeliveryId` are value objects accessed only through `Delivery`. `Location` is shared with the truck module. `DistanceCalculator` is a domain service.

```mermaid
classDiagram
    class Delivery {
        <<aggregate root>>
        +DeliveryId id
        +TruckId truckId
        +String shipmentId
        +Location origin
        +Location destination
        +int totalDays
        +int remainingDays
        +List~DeliveryItem~ items
        +advance(days) DomainEvent
        +isArrived() boolean
    }
    class DeliveryItem {
        <<value object>>
        +String materialType
        +int quantity
    }
    class DeliveryId {
        <<value object>>
        +String value
    }
    Delivery --> DeliveryId
    Delivery --> TruckId
    Delivery "1" --> "*" DeliveryItem
```

---

## Domain services

```mermaid
classDiagram
    class OptimalTruckSelector {
        <<domain service>>
        +findBestTruck(origin, destination, itemCount, trucks) Truck
    }
    class DistanceCalculator {
        <<domain service>>
        +calculate(a, b) int
        +isOnRoute(point, from, to) boolean
    }
```

---

## Use cases

### UC1 — shipment.requested.v1 received → Assign truck

```mermaid
flowchart TD
    UC1([shipment.requested.v1 received])
    UC1 --> A1[AssignTruck use case]
    A1 --> A2[OptimalTruckSelector.findBestTruck\norigin / destination / itemCount / all trucks]
    A2 --> A3{Truck found?}
    A3 -->|NO| A9[No truck available — discard or retry]
    A3 -->|YES AVAILABLE| A4[Create Delivery\nSet Truck status to IN_TRANSIT\nSet currentLoad = itemCount]
    A3 -->|YES IN_TRANSIT passing by| A5[Create Delivery\nAdd to truck.deliveryIds\nUpdate currentLoad += itemCount]
    A4 --> A6[Publish truck.status.changed.v1\nreason: DISPATCHED\ncurrentLoad / capacity]
    A5 --> A7[Publish truck.status.changed.v1\nreason: LOAD_UPDATED\ncurrentLoad / capacity]
```

### UC2 — simulation.time.tick received → Advance trucks

```mermaid
flowchart TD
    UC2([simulation.time.tick received])
    UC2 --> B1[AdvanceTrucks use case]
    B1 --> B2[Calculate daysAdvanced = currentDay - lastDay]
    B2 --> B3[For each IN_TRANSIT Delivery: advance]
    B3 --> B4{isArrived?}
    B4 -->|YES| B5[Publish delivery.completed.v1\nUpdate truck currentLoad -= delivery.itemCount\nTruck.completeDelivery]
    B5 --> B6{truck.deliveryIds empty?}
    B6 -->|YES| B7[Set Truck to AVAILABLE\nPublish truck.status.changed.v1\nreason: RETURNED_TO_BASE\ncurrentLoad: 0]
    B6 -->|NO| B8[Truck stays IN_TRANSIT\nPublish truck.status.changed.v1\nreason: LOAD_UPDATED\ncurrentLoad / capacity]
    B4 -->|NO| B9[Publish truck.position.updated.v1]
```

### UC3 — Register truck (REST)

```mermaid
flowchart TD
    UC3([POST /trucks])
    UC3 --> C1[RegisterTruck use case]
    C1 --> C2[Create Truck\nstatus: AVAILABLE]
    C2 --> C3[Publish truck.status.changed.v1\nreason: TRUCK_REGISTERED\noldStatus: null]
```

---

## Events published

| Event | Trigger | Consumed by |
|---|---|---|
| truck.status.changed.v1 | Register, dispatch, load updated, delivery, return | Reporting |
| truck.position.updated.v1 | Each tick while IN_TRANSIT | Map (UI), Reporting |
| delivery.completed.v1 | Truck arrives at a delivery destination | Warehouses, Reporting, Map (UI) |

---

## Events consumed

| Event | Emitted by | Use case triggered |
|---|---|---|
| shipment.requested.v1 | Warehouses / Factories | AssignTruck |
| simulation.time.tick | Ruben (Simulation) | AdvanceTrucks |

---

