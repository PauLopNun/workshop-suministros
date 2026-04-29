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
        +int simulationDay
        +int daysAdvanced
        +String timestamp
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

    UC2([GET /map]) --> B1[GetMapState]
    B1 --> B2[Returns current positions of all elements]

    EV1([truck.assigned.v1 received]) --> C1[Updates TruckPosition in MapState]
    EV2([delivery.completed.v1 received]) --> C2[Sets truck status to IDLE]
    EV3([factory.registered.v1 received]) --> C3[Adds factory to map]
    EV4([warehouse.registered.v1 received]) --> C4[Adds warehouse to map]
```

## Package structure

```
time-service/
├── simulation-clock/
│   ├── domain/
│   │   ├── SimulationClock.java
│   │   ├── SimulationDay.java
│   │   └── event/TimeAdvancedEvent.java
│   ├── application/usecase/AdvanceTimeUseCase.java
│   └── infrastructure/
│       ├── rest/TickController.java
│       └── persistence/SimulationClockJpaRepository.java
└── map-state/
    ├── domain/
    │   ├── MapState.java
    │   ├── TruckPosition.java
    │   ├── FactoryPosition.java
    │   ├── WarehousePosition.java
    │   └── Location.java
    ├── application/usecase/GetMapStateUseCase.java
    └── infrastructure/
        ├── rest/MapController.java
        └── messaging/MapStateEventListener.java
```
