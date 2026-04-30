# Transport — Sergi

Manages the truck fleet and deliveries between warehouses.
Moves trucks on each time tick and notifies when a delivery is completed.

## Modules

### Module: truck

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
        +String currentDeliveryId
        +assignDelivery(delivery)
        +advance(days) DomainEvent
        +completeDelivery() DomainEvent
        +isAvailable() boolean
        +distanceTo(location) int
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
        IDLE
        IN_TRANSIT
    }
    Truck --> TruckId
    Truck --> Location
    Truck --> TruckStatus
```

### Module: delivery

```mermaid
classDiagram
    class Delivery {
        <<aggregate root>>
        +DeliveryId id
        +String truckId
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
        +String productId
        +int quantity
    }
    class DeliveryId {
        <<value object>>
        +String value
    }
    Delivery --> DeliveryId
    Delivery "1" --> "*" DeliveryItem
```

## Domain services

```mermaid
classDiagram
    class OptimalTruckSelector {
        <<domain service>>
        +findBestTruck(origin, items) Truck
    }
    class DistanceCalculator {
        <<domain service>>
        +calculate(a, b) int
    }
```

## Use cases

```mermaid
flowchart TD
    UC1([dispatch.requested.v1 received]) --> A1[AssignTruck]
    A1 --> A2[OptimalTruckSelector.findBestTruck]
    A2 --> A3[Creates Delivery\nSets Truck to IN_TRANSIT]
    A3 --> A4[Publishes truck.assigned.v1]

    UC2([time.advanced.v1 received]) --> B1[AdvanceTrucks]
    B1 --> B2[For each IN_TRANSIT Delivery advance]
    B2 --> B3{isArrived?}
    B3 -->|YES| B4[Publishes delivery.completed.v1\nTruck back to IDLE]
    B3 -->|NO| B5[Publishes truck.position.updated.v1]
```

## Events published

| Event | Consumed by |
|---|---|
| truck.registered.v1 | Time/Map, Reporting |
| truck.assigned.v1 | Time/Map, Reporting |
| truck.position.updated.v1 | Time/Map |
| delivery.created.v1 | Reporting |
| delivery.completed.v1 | Production, Warehouse, Reporting |
