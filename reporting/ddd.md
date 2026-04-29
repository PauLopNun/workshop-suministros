# Reporting — Pedro (Supporting Subdomain)

Consumes all events from all services and builds read-only projections.
Does not publish any event. Does not modify any data.

## Modules

### Module: event-log

```mermaid
classDiagram
    class EventLog {
        <<aggregate root>>
        +EventLogId id
        +String eventType
        +String sourceService
        +String payload
        +int simulationDay
        +String occurredAt
    }
    class EventLogId {
        <<value object>>
        +String value
    }
    EventLog --> EventLogId
```

### Module: blocked-order

```mermaid
classDiagram
    class BlockedOrder {
        <<aggregate root>>
        +String orderId
        +String factoryId
        +String reason
        +int blockedSinceDay
    }
```

## Read models

```mermaid
classDiagram
    class OrderHistoryProjection {
        <<read model>>
        +String orderId
        +String factoryId
        +String status
        +int simulationDay
    }
    class SystemStatsProjection {
        <<read model>>
        +int totalOrders
        +int completedOrders
        +int blockedOrders
        +int trucksInTransit
    }
    class BlockedOrdersProjection {
        <<read model>>
        +String orderId
        +String reason
        +int blockedSinceDay
    }
```

## Events consumed

| Event | Source | Action |
|---|---|---|
| time.advanced.v1 | Time | Saves to EventLog |
| truck.registered.v1 | Transport | Saves to EventLog |
| truck.assigned.v1 | Transport | Saves to EventLog, updates stats |
| truck.position.updated.v1 | Transport | Saves to EventLog |
| delivery.created.v1 | Transport | Saves to EventLog |
| delivery.completed.v1 | Transport | Saves to EventLog, updates stats |
| production.order.created.v1 | Production | Saves to EventLog, updates history |
| production.order.started.v1 | Production | Saves to EventLog, updates history |
| production.order.blocked.v1 | Production | Creates BlockedOrder, saves to EventLog |
| production.order.completed.v1 | Production | Saves to EventLog, updates history |
| replenishment.requested.v1 | Warehouse | Saves to EventLog |
| warehouse.stock.changed.v1 | Warehouse | Saves to EventLog, updates stats |

## Package structure

```
reporting-service/
├── event-log/
│   ├── domain/
│   │   ├── EventLog.java
│   │   └── EventLogId.java
│   ├── application/usecase/HandleAnyEvent.java
│   └── infrastructure/messaging/AllEventsListener.java
└── blocked-order/
    ├── domain/BlockedOrder.java
    ├── application/usecase/
    │   ├── GetBlockedOrders.java
    │   ├── GetOrderHistory.java
    │   └── GetSystemStats.java
    └── infrastructure/rest/ReportingController.java
```
