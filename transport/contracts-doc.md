# Messaging Contracts - Transport Service

**Supply Chain Workshop | GFT | May 2026**
MVP Version | without BREAKDOWN

---

## Legend

| Symbol | Meaning |
|---|---|
| `[+]` | Field added on top of the original design |
| `[-] MVP` | Removed in this version, to be recovered when BREAKDOWN is implemented |
| **Value Object** | No own identity. Identified solely by its attribute values. |

---

## RabbitMQ Topology

Transport declares the exchanges and its own input queues. Consumer services must declare their own queues and bindings.

| Exchange | Routing key | Direction | Transport queue |
|---|---|---|---|
| `trucks.exchange` | `truck.registered.v1` | Publishes | N/A |
| `trucks.exchange` | `truck.status.changed.v1` | Publishes | N/A |
| `trucks.exchange` | `truck.position.updated.v1` | Publishes | N/A |
| `shipments.exchange` | `delivery.completed.v1` | Publishes | N/A |
| `shipments.exchange` | `shipment.requested.v1` | Consumes | `trucks.shipment.requested` |
| `simulation.exchange` | `time.advanced.v1` | Consumes | `trucks.time.advanced` |

---

## Published Contracts

The Transport Service emits these events to RabbitMQ.

---

### `truck.registered.v1`

Emitted when a new truck is registered via `POST /trucks`.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Identifier of the new truck |
| `name` | String | Display name of the truck |
| `location` | Value Object | Starting position on the grid |
| `location.x` | Integer | X coordinate |
| `location.y` | Integer | Y coordinate |
| `capacity` | Integer | Maximum number of items the truck can carry |
| `timestamp` | Integer | Simulation day of registration. Current implementation emits `0` on REST registration |

**Published on:** `trucks.exchange` with routing key `truck.registered.v1`
**Consumers:** Reporting | Map UI

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Truck 01",
  "location": { "x": 0, "y": 0 },
  "capacity": 10,
  "timestamp": 0
}
```

---

### `truck.status.changed.v1`

Notifies a change in a truck's operational status or load.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Truck whose status has changed |
| `oldStatus` | Enum/null | Previous status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED`. `null` on initial registration |
| `newStatus` | Enum | New status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |
| `position` | Value Object | Position at the moment of the change |
| `position.x` | Integer | X coordinate |
| `position.y` | Integer | Y coordinate |
| `currentLoad` | Integer | Number of items the truck is currently carrying |
| `capacity` | Integer | Maximum number of items the truck can carry |
| `timestamp` | Integer | Simulation day of the status change |
| `reason` | String | Change context for Reporting |
| `reasonCode` | Enum | `[-] MVP` - typed enum to be implemented with BREAKDOWN |

**Published on:** `trucks.exchange` with routing key `truck.status.changed.v1`
**Consumers:** Reporting

**Current reasons:** `TRUCK_REGISTERED`, `DISPATCHED`, `LOAD_UPDATED`, `RETURNED_TO_BASE`

The current implementation does not publish a `DELIVERED` transition. When the last delivery is completed, the truck moves directly from `IN_TRANSIT` to `AVAILABLE` with `reason: RETURNED_TO_BASE`.

Example - initial registration:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": null,
  "newStatus": "AVAILABLE",
  "position": { "x": 0, "y": 0 },
  "currentLoad": 0,
  "capacity": 10,
  "timestamp": 0,
  "reason": "TRUCK_REGISTERED"
}
```

Example - dispatch to shipment:

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

Example - additional load or remaining load update:

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

Example - all deliveries completed:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "oldStatus": "IN_TRANSIT",
  "newStatus": "AVAILABLE",
  "position": { "x": 8, "y": 2 },
  "currentLoad": 0,
  "capacity": 10,
  "timestamp": 7,
  "reason": "RETURNED_TO_BASE"
}
```

---

### `truck.position.updated.v1`

Truck movement event. It carries only map position data.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Identifies the truck to move on the map |
| `location` | Value Object | Current position |
| `location.x` | Integer | X coordinate on the grid |
| `location.y` | Integer | Y coordinate on the grid |

**Published on:** `trucks.exchange` with routing key `truck.position.updated.v1`
**Consumers:** Map UI | Reporting

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "location": { "x": 5, "y": 3 }
}
```

---

### `delivery.completed.v1`

Confirms that a truck completed a delivery at its destination.

| Field | Type | Notes |
|---|---|---|
| `shipmentId` | String (UUID) | Correlates the completed delivery with the original shipment request |
| `truckId` | String (UUID) | Truck that completed the delivery |
| `items[]` | Array of Value Objects | Delivered items |
| `items[].materialType` | String | Type of material delivered |
| `items[].quantity` | Integer | Quantity delivered |
| `location` | Value Object | Destination location |
| `location.x` | Integer | X coordinate of the destination |
| `location.y` | Integer | Y coordinate of the destination |
| `completedAt` | Integer | Simulation day on which the delivery was completed |

**Published on:** `shipments.exchange` with routing key `delivery.completed.v1`
**Consumers:** Warehouses | Reporting | Map UI

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

## REST Endpoints

---

### `POST /trucks`

Registers a new truck in the fleet.

**Side effects:** Publishes `truck.registered.v1` and `truck.status.changed.v1` with `reason: TRUCK_REGISTERED`.

Request:

| Field | Type | Notes |
|---|---|---|
| `name` | String | Required, non-blank |
| `x` | Integer | Starting X coordinate |
| `y` | Integer | Starting Y coordinate |
| `capacity` | Integer | Required, minimum `1` |

```json
{
  "name": "Truck 01",
  "x": 0,
  "y": 0,
  "capacity": 10
}
```

Response `201`:

```json
{
  "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Truck 01",
  "location": { "x": 0, "y": 0 },
  "status": "AVAILABLE"
}
```

---

### `GET /trucks`

Returns the full list of registered trucks with current position and status. Map UI uses it on startup before receiving live events.

| Field | Type | Notes |
|---|---|---|
| `truckId` | String (UUID) | Truck identifier |
| `name` | String | Truck display name |
| `location` | Value Object | Current position on the grid |
| `location.x` | Integer | X coordinate |
| `location.y` | Integer | Y coordinate |
| `status` | Enum | Current status: `AVAILABLE` \| `IN_TRANSIT` \| `DELIVERED` |

```json
[
  {
    "truckId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "Truck 01",
    "location": { "x": 3, "y": 5 },
    "status": "IN_TRANSIT"
  }
]
```

---

## Consumed Contracts

The Transport Service owns queues for these RabbitMQ events.

---

### `time.advanced.v1`

System time event. The current implementation ignores unknown fields, skips messages with `daysAdvanced <= 0`, and advances trucks by one grid step per simulated day.

| Field | Type | Notes |
|---|---|---|
| `previousDayNumber` | Integer | Previous simulation day |
| `currentDayNumber` | Integer | Current simulation day. Used as `completedAt` and status-change timestamp |
| `daysAdvanced` | Integer | Number of days to process |

**Consumed from:** `simulation.exchange` with routing key `time.advanced.v1` through queue `trucks.time.advanced`
**Emitter:** Simulation Service

```json
{
  "previousDayNumber": 2,
  "currentDayNumber": 3,
  "daysAdvanced": 1
}
```

---

### `shipment.requested.v1`

Request to transport items between two grid points. The RabbitMQ binding exists in Transport, and the application use case expects this data shape. The current `DispatchRequestedListener` source file is still empty, so the listener adapter needs implementation before the event is processed end to end.

| Field | Type | Notes |
|---|---|---|
| `shipmentId` | String (UUID) | Allows correlation with `delivery.completed.v1` |
| `origin` | Value Object | Pickup location |
| `origin.x` | Integer | Origin X coordinate |
| `origin.y` | Integer | Origin Y coordinate |
| `destination` | Value Object | Delivery destination |
| `destination.x` | Integer | Destination X coordinate |
| `destination.y` | Integer | Destination Y coordinate |
| `items[]` | Array of Value Objects | Items to transport |
| `items[].materialType` | String | Type of material to transport |
| `items[].quantity` | Integer | Quantity to transport |
| `requestedAt` | Integer | Simulation day on which the request was made |

**Consumed from:** `shipments.exchange` with routing key `shipment.requested.v1` through queue `trucks.shipment.requested`
**Emitter:** Warehouses / Factories

**Action:** Select the closest available truck with enough remaining capacity. If none is available, try an `IN_TRANSIT` truck that still has capacity. Create a `Delivery`, update truck load/status, and publish `truck.status.changed.v1`.

```json
{
  "shipmentId": "c9d8e7f6-a5b4-3210-9876-543210fedcba",
  "origin": { "x": 0, "y": 0 },
  "destination": { "x": 8, "y": 2 },
  "items": [
    { "materialType": "wood", "quantity": 6 },
    { "materialType": "nails", "quantity": 12 }
  ],
  "requestedAt": 3
}
```

---

## Summary

| Event | Direction | Exchange | Consumers / Emitter |
|---|---|---|---|
| `truck.registered.v1` | Publishes | `trucks.exchange` | Reporting \| Map UI |
| `truck.status.changed.v1` | Publishes | `trucks.exchange` | Reporting |
| `truck.position.updated.v1` | Publishes | `trucks.exchange` | Map UI \| Reporting |
| `delivery.completed.v1` | Publishes | `shipments.exchange` | Warehouses \| Reporting \| Map UI |
| `time.advanced.v1` | Consumes | `simulation.exchange` | Emitted by Simulation |
| `shipment.requested.v1` | Consumes | `shipments.exchange` | Emitted by Warehouses / Factories |

---

## Note on Value Objects

The `location`, `origin`, `destination`, `position`, and `items[]` fields are Value Objects:

- They carry no ID field.
- They are copied in full in each message and are not referenced by ID.
- `items[]` structure is `{ "materialType": String, "quantity": Integer }`.
