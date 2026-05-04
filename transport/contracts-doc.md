# Messaging Contracts — Transport Service

**Supply Chain Workshop | GFT | April 2026**  
MVP Version | without BREAKDOWN

---

## Legend

| Symbol | Meaning |
|---|---|
| `[+]` | Field added on top of the original design |
| `[-] MVP` | Removed in this version, to be recovered when BREAKDOWN is implemented |
| **Value Object** | No own identity (no ID). Identified solely by its attribute values. |

---

## PUBLISHED Contracts

The Transport Service emits these events to RabbitMQ.

---

### `truck.registered.v1`

Emitted when a new truck is registered via `POST /trucks`. Explicit registration event — Reporting listens to this to know a new truck has entered the fleet.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Identifier of the new truck |
| `name` | String | Display name of the truck |
| `position` | Value Object | Starting position on the grid |
| `position.x` | Number | X coordinate |
| `position.y` | Number | Y coordinate |
| `capacity` | Integer | Maximum number of DeliveryItems the truck can carry |
| `timestamp` | Integer | Simulation day of registration |

**Consumers:** Reporting | Map (UI)

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Truck 01",
  "position": { "x": 0, "y": 0 },
  "capacity": 10,
  "timestamp": 1
}
```

---

### `truck.position.updated.v1`

Real-time truck position. Designed for high frequency: carries no business data to avoid saturating the network.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Identifies the truck to move on the map |
| `location` | Value Object | Value Object: no ID. Identified by its coordinates |
| `location.x` | Number | X coordinate on the grid |
| `location.y` | Number | Y coordinate on the grid |

**Consumers:** Map (UI) | Reporting

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "location": { "x": 5, "y": 3 }
}
```

---

### `delivery.completed.v1`

Confirms that a truck has completed a delivery at a destination warehouse.

| Field | Type | Notes |
|---|---|---|
| `shipmentId` | String (UUID) | Order ID. Used by the receiver to mark the delivery as received |
| `truckId` | String (UUID) | `[+]` Identifies the truck that completed the delivery. Used by Map (UI) to remove or update the truck icon |
| `items[]` | Array of Value Objects | Value Object: no ID. Identified by their attributes |
| `items[].materialType` | String | Type of material delivered (agree naming with the team) |
| `items[].quantity` | Number | Quantity delivered |
| `location` | Value Object | Value Object: no ID. Identified by its coordinates |
| `location.x` | Number | X coordinate of the destination |
| `location.y` | Number | Y coordinate of the destination |
| `completedAt` | Integer | `[+]` Simulation day on which the delivery was completed. Required to calculate transit times in Reporting |

**Consumers:** Warehouses | Reporting | Map (UI)

> The `location` field is exclusively for Reporting (log verification and traceability). Warehouses can ignore it — they already know their own location.
>
> Map (UI) only needs `truckId` from this event — to remove or update the truck icon when the delivery is completed. All other fields can be ignored by Map (UI).

```json
{
  "shipmentId": "f7e6d5c4-b3a2-1098-fedc-ba9876543210",
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "items": [
    { "materialType": "wood", "quantity": 6 },
    { "materialType": "nails", "quantity": 12 }
  ],
  "location": { "x": 8, "y": 2 },
  "completedAt": 5
}
```

---

### `truck.status.changed.v1`

Notifies a change in a truck's operational status. Allows Reporting to trace the complete lifecycle after registration.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Truck whose status has changed |
| `oldStatus` | Enum | Previous status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |
| `newStatus` | Enum | New status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |
| `position` | Value Object | Value Object: no ID. Identified by its coordinates |
| `position.x` | Number | X coordinate at the moment of the change |
| `position.y` | Number | Y coordinate at the moment of the change |
| `currentLoad` | Integer | `[+]` Number of DeliveryItems the truck is currently carrying |
| `capacity` | Integer | `[+]` Maximum number of DeliveryItems the truck can carry |
| `timestamp` | Integer | Simulation day of the status change |
| `reason` | String | Change context for Reporting. E.g.: `DISPATCHED`, `LOAD_UPDATED`, `RETURNED_TO_BASE` |
| `reasonCode` | Enum | `[-] MVP` — Typed enum to be implemented with BREAKDOWN |

**Consumers:** Reporting

**Status cycle:** `AVAILABLE` -> `IN_TRANSIT` -> `DELIVERED` -> `AVAILABLE`

**Multi-delivery:** When a truck already `IN_TRANSIT` accepts an additional shipment on its route and within capacity, this event is published with `reason: LOAD_UPDATED` and the status unchanged (`IN_TRANSIT`). When a truck completes one delivery but still carries others, it also publishes `reason: LOAD_UPDATED` with the updated `currentLoad`.

Example — dispatch to shipment:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": "AVAILABLE",
  "newStatus": "IN_TRANSIT",
  "position": { "x": 0, "y": 0 },
  "currentLoad": 6,
  "capacity": 10,
  "timestamp": 3,
  "reason": "DISPATCHED"
}
```

Example — additional load picked up while in transit:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": "IN_TRANSIT",
  "newStatus": "IN_TRANSIT",
  "position": { "x": 3, "y": 2 },
  "currentLoad": 9,
  "capacity": 10,
  "timestamp": 4,
  "reason": "LOAD_UPDATED"
}
```

Example — truck returns to base after all deliveries:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": "IN_TRANSIT",
  "newStatus": "AVAILABLE",
  "position": { "x": 0, "y": 0 },
  "currentLoad": 0,
  "capacity": 10,
  "timestamp": 7,
  "reason": "RETURNED_TO_BASE"
}
```

---

## REST Endpoints

---

### `POST /trucks`

Registers a new truck in the fleet. Publishes `truck.registered.v1` after creation.

**Consumers:** Internal / Admin

---

### `GET /trucks`

Returns the full list of registered trucks with their current position and status. Designed for Map (UI) to fetch the initial state on startup — before any events are received.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Truck identifier |
| `name` | String | Truck display name |
| `location` | Value Object | Current position on the grid |
| `location.x` | Number | X coordinate |
| `location.y` | Number | Y coordinate |
| `status` | Enum | Current status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |

**Consumers:** Map (UI)

> Map (UI) calls this endpoint on startup to render the initial truck positions. From that point on, it stays updated via `truck.position.updated.v1` (movement) and `delivery.completed.v1` (arrival).

```json
[
  {
    "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "Truck 01",
    "location": { "x": 3, "y": 5 },
    "status": "IN_TRANSIT"
  },
  {
    "truckId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "name": "Truck 02",
    "location": { "x": 0, "y": 0 },
    "status": "AVAILABLE"
  }
]
```

---

## CONSUMED Contracts

The Transport Service listens to these events from RabbitMQ.

---

### `time.advanced.v1`

System time engine. The Simulation service sends the current day as an integer. The Transport service calculates internally how many days have advanced by comparing against the last received day.

| Field | Type | Notes |
|---|---|---|
| `currentDay` | Integer | Current simulation day. Starts at 1 on the first tick |

**Emitter:** Ruben (Simulation Service)

**Action:** Calculate `daysAdvanced = currentDay - lastDay`. For each `IN_TRANSIT` truck apply: `P_new = P_current + (speed * daysAdvanced * direction)`

```json
{
  "currentDay": 3
}
```

---

### `shipment.requested.v1` [NEW]

Request to transport materials between two points. This contract covers the identified gap: it triggers the internal logic that finds the optimal available truck and assigns it.

| Field | Type | Notes |
|---|---|---|
| `shipmentId` | String (UUID) | Allows correlation with `delivery.completed.v1` |
| `originId` | String | ID of the origin warehouse or factory |
| `destinationId` | String | ID of the destination warehouse |
| `items[]` | Array of Value Objects | Value Object: no ID. Identified by their attributes |
| `items[].materialType` | String | Type of material to transport |
| `items[].quantity` | Number | Quantity to transport |
| `requestedAt` | Integer | Simulation day on which the request was made |

**Emitter:** Warehouses / Factories

**Action:** Find optimal available truck -> assign it -> mark `IN_TRANSIT` -> publish `truck.status.changed.v1`

```json
{
  "shipmentId": "c9d8e7f6-a5b4-3210-9876-543210fedcba",
  "originId": "warehouse-north-01",
  "destinationId": "warehouse-south-03",
  "items": [
    { "materialType": "wood", "quantity": 6 },
    { "materialType": "nails", "quantity": 12 }
  ],
  "requestedAt": 3
}
```

---

## Summary

| Event | Direction | Consumers / Emitter |
|---|---|---|
| `truck.registered.v1` | PUBLISHES | Reporting \| Map (UI) |
| `truck.position.updated.v1` | PUBLISHES | Map (UI) \| Reporting |
| `delivery.completed.v1` | PUBLISHES | Warehouses \| Reporting \| Map (UI) |
| `truck.status.changed.v1` | PUBLISHES | Reporting |
| `time.advanced.v1` | CONSUMES | Emitted by: Ruben (Simulation) |
| `shipment.requested.v1` [NEW] | CONSUMES | Emitted by: Warehouses / Factories |

---

## Note on Value Objects

The `items[]` fields in `delivery.completed.v1` and `shipment.requested.v1` are arrays of Value Objects, not entities. This means:

- They carry no ID field (no `itemId`). They are identified by the value of their attributes.
- They are immutable: they are copied in full in each message and never referenced by ID.
- The structure is: `{ materialType: String, quantity: Number }`.
- The field name `materialType` must be agreed upon with the rest of the team so that all services use the same material naming convention.
