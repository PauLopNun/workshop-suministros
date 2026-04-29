# Time + Map — Ruben

Upstream of all services. Publishes `time.advanced.v1` when the user advances time.
Builds the map by listening to registration events from other services.

## Modules

### Module: simulation-clock

```mermaid
classDiagram
    class SimulationClock {
        <<aggregate root>>
        +SimulationDay currentDay
        +advanceDay(days) DomainEvent[]
        +getCurrentDay() SimulationDay
    }
    class SimulationDay {
        <<value object>>
        +int value
    }

    class TimeAdvancedEvent {
    <<domain event>>
    +UUID eventId
    +int previousDay
    +int currentDay
    +int daysAdvanced
    +Instant occurredAt
}
    SimulationClock --> SimulationDay
    SimulationClock ..> TimeAdvancedEvent : publishes
```

### Module: map-state

```mermaid
classDiagram
    class MapState {
        <<aggregate root>>
        +updateTruckPosition(truckId, location)
        +registerFactory(factoryId, name, location)
        +registerWarehouse(warehouseId, name, location)
    }
    class TruckPosition {
        <<value object>>
        +String truckId
        +Location location
        +String status
        +Location destLocation
    }
    class FactoryPosition {
        <<value object>>
        +String factoryId
        +String name
        +Location location
    }
    class WarehousePosition {
        <<value object>>
        +String warehouseId
        +String name
        +Location location
    }
    class Location {
        <<value object>>
        +int x
        +int y
    }
    MapState "1" --> "*" TruckPosition
    MapState "1" --> "*" FactoryPosition
    MapState "1" --> "*" WarehousePosition
```

## Use cases

```mermaid
flowchart TD
    UC1([POST /tick]) --> A1[AdvanceTime]
    A1 --> A2[SimulationClock.advanceDay]
    A2 --> A3[Publishes time.advanced.v1]
    A3 --> A4[Payload: previousDay, currentDay, daysAdvanced]

    UC2([GET /map]) --> B1[GetMapState]
    B1 --> B2[Returns current positions of all elements]

    EV1([truck.assigned.v1 received]) --> C1[Updates TruckPosition in MapState]
    EV2([delivery.completed.v1 received]) --> C2[Sets truck status to IDLE]
    EV3([factory.registered.v1 received]) --> C3[Adds factory to map]
    EV4([warehouse.registered.v1 received]) --> C4[Adds warehouse to map]
```

## Contracts with other microservices

| Microservice | Publishes | Consumes | Purpose |
|---|---|---|---|
| **Transport** | `time.advanced.v1` | `truck.registered.v1`<br>`truck.assigned.v1`<br>`truck.position.updated.v1`<br>`delivery.completed.v1` | Transport uses the time advance event to move trucks. Time + Map uses Transport events to display trucks on the map and update their positions. |
| **Production** | `time.advanced.v1` | `factory.registered.v1`<br>`factory.updated.v1` *(optional)* | Production uses the time advance event to progress production orders. Time + Map uses Production events to display factories on the map. |
| **Warehouse** | `time.advanced.v1` | `warehouse.registered.v1`<br>`warehouse.updated.v1` *(optional)* | Warehouse may use the time advance event to check stock, consumption or replenishment needs. Time + Map uses Warehouse events to display warehouses on the map. |
| **Reporting** | `time.advanced.v1` | None | Reporting records time advances for history, monitoring and statistics. |

