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
  "location": { "x": 8, "y": 2 }
}
```

---

### `delivery.completed.v1`

Confirms that a truck has completed a delivery at a destination warehouse.

| Field | Type | Notes |
|---|---|---|
| `shipmentId` | String (UUID) | Order ID. Used by the receiver to mark the delivery as received |
| `truckId` | String (UUID) | `[+]` Identifies the truck that completed the delivery. Used by Map (UI) to remove or update the truck icon, and by Reporting to correlate with `truck.status.changed.v1` |
| `items[]` | Array of Value Objects | Value Object: no ID. Identified by their attributes |
| `items[].materialType` | String | Type of material delivered (agree naming with the team) |
| `items[].quantity` | Number | Quantity delivered |
| `location` | Value Object | Value Object: no ID. Identified by its coordinates |
| `location.x` | Number | X coordinate of the destination |
| `location.y` | Number | Y coordinate of the destination |
| `completedAt` | ISO-8601 | `[+]` Exact delivery timestamp. Required to calculate transit times in Reporting |

**Consumers:** Warehouses | Reporting

> The `location` field is exclusively for Reporting (log verification and traceability). Warehouses can ignore it — they already know their own location.
> 
> Map (UI) only needs `truckId` from this event — to remove or update the truck icon when the delivery is completed. All other fields can be ignored by Map (UI).


```json
{
  "shipmentId": "f7e6d5c4-b3a2-1098-fedc-ba9876543210",
  "items": [
    { "materialType": "wood", "quantity": 6 },
    { "materialType": "nails", "quantity": 12 }
  ],
  "location": { "x": 8, "y": 2 },
  "completedAt": "2026-04-29T10:34:00Z"
}
```

---

### `truck.status.changed.v1`

Notifies a change in a truck's status. Allows Reporting to trace the complete lifecycle.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Truck whose status has changed |
| `oldStatus` | Enum \| null | Previous status. `null` on the truck's initial registration |
| `newStatus` | Enum | New status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |
| `position.x` | Number | X coordinate at the moment of the change |
| `position.y` | Number | Y coordinate at the moment of the change |
| `timestamp` | ISO-8601 | Timestamp of the status change |
| `reason` | String | Change context for Reporting. E.g.: `TRUCK_REGISTERED`, `DISPATCHED`, `DELIVERED`, `RETURNED_TO_BASE` |
| `reasonCode` | Enum | `[-] MVP` — Typed enum to be implemented with BREAKDOWN |

**Consumers:** Reporting

**Status cycle:** `AVAILABLE` -> `IN_TRANSIT` -> `DELIVERED` -> `AVAILABLE`

**Truck registration:** There is no separate registration event. When a truck is registered, this same event is emitted with `newStatus: AVAILABLE` and `oldStatus: null`. Reporting detects the registration by interpreting the first event with `oldStatus null` for a given `truckId`.

Example — truck registration:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": null,
  "newStatus": "AVAILABLE",
  "position": { "x": 0, "y": 0 },
  "timestamp": "2026-04-29T08:00:00Z",
  "reason": "TRUCK_REGISTERED"
}
```

Example — dispatch to shipment:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": "AVAILABLE",
  "newStatus": "IN_TRANSIT",
  "position": { "x": 0, "y": 0 },
  "timestamp": "2026-04-29T09:15:00Z",
  "reason": "DISPATCHED"
}
```

---

## CONSUMED Contracts

The Transport Service listens to these events from RabbitMQ.

---

### `simulation.time.tick`

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
| `requestedAt` | ISO-8601 | Timestamp when the request was made |

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
  "requestedAt": "2026-04-29T09:10:00Z"
}
```

---

## Summary

| Event | Direction | Consumers / Emitter |
|---|---|---|
| `truck.position.updated.v1` | PUBLISHES | Map (UI) \| Reporting |
| `delivery.completed.v1` | PUBLISHES | Warehouses \| Reporting |
| `truck.status.changed.v1` | PUBLISHES | Reporting |
| `simulation.time.tick` | CONSUMES | Emitted by: Ruben (Simulation) |
| `shipment.requested.v1` [NEW] | CONSUMES | Emitted by: Warehouses / Factories |

---

## Note on Value Objects

The `items[]` fields in `delivery.completed.v1` and `shipment.requested.v1` are arrays of Value Objects, not entities. This means:

- They carry no ID field (no `itemId`). They are identified by the value of their attributes.
- They are immutable: they are copied in full in each message and never referenced by ID.
- The structure is: `{ materialType: String, quantity: Number }`.
- The field name `materialType` must be agreed upon with the rest of the team so that all services use the same material naming convention.
