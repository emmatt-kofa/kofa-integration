# Field Executor Movement — Mobile Integration Guide

> **Audience:** Swap & Go mobile (field executor flows)  
> **Base path:** `/api/v1/sg/rider`  
> **Auth:** Sanctum — user must be a field executor

---

## Terminology

| Term | Meaning |
|------|---------|
| **Firm trip** (`type: "firm"`) | Specific batteries are pre-assigned on the trip. See `pre_assigned_batteries` on the trip resource. |
| **Floating trip** (`type: "floating"`) | Only a count is known (`requested_count`). The rider chooses which batteries/slots at pickup time. Trip items are created on `/pickup`. |
| **Swap station source/destination** | IoT lend/return flow applies (`unlock-source`, `unlock-destination`). |
| **Storage location source/destination** | No slot unlock. Rider handles batteries physically; confirm via `/pickup` or `/deliver` only. |
| **Custodian depot destination** | Special storage depot. `/deliver` records arrival only; batteries stay in transit until a depot rep confirms receipt. |

---

## Trip fields the app needs

From `MovementTripResource`:

| Field | Use |
|-------|-----|
| `type` | `"firm"` or `"floating"` — drives which flows apply |
| `trip_status` | Current phase (`accepted`, `picked_up`, `in_transit`, `delivered`, …) |
| `requested_count` | Batteries required (especially for floating) |
| `source.type` | `"swap_station"` or `"storage_location"` |
| `destination.type` | `"swap_station"` or `"storage_location"` |
| `pre_assigned_batteries` | Firm only — list of `{ id, physical_id }`; `null` for floating |
| `picked_up_batteries` | Batteries already picked up on this trip (after `/pickup`) |

---

## Which endpoints apply?

Use this matrix before building a screen:

### Source phase (`trip_status === "accepted"`)

| Source | Firm | Floating |
|--------|------|----------|
| **Swap station** | `source-batteries` → `unlock-source` (no `slot_ids`) → poll → optional `source-pickup-progress` → `pickup` | `source-batteries` → `unlock-source` (**`slot_ids` required**) → poll → **`source-pickup-progress`** → `pickup` |
| **Storage location** | `source-batteries` → `pickup` (no unlock) | `source-batteries` → `pickup` with chosen `battery_ids` (no unlock) |

### Destination phase (`trip_status === "picked_up"` or `"in_transit"`)

| Destination | Firm | Floating |
|-------------|------|----------|
| **Swap station** | `destination-slots` → `unlock-destination` (per battery) → poll → optional `destination-delivery-progress` → `deliver` | `destination-slots` → `unlock-destination` (per battery) → poll → **`destination-delivery-progress`** → `deliver` |
| **Storage location** (non-depot) | `deliver` directly (no unlock) | `deliver` directly (no unlock) |
| **Custodian depot** | `deliver` → marks arrival only (`arrived_at`); trip stays `in_transit` | Same |

### Progress endpoints (swap station IoT only)

| Endpoint | When to call |
|----------|--------------|
| `GET .../source-pickup-progress` | Source is swap station **and** trip is `accepted` |
| `GET .../destination-delivery-progress` | Destination is swap station **and** trip is `picked_up` or `in_transit` |

**Backend enforcement note:**

- **Floating + swap station source:** `/pickup` requires `can_confirm_pickup === true` (completed lends match `requested_count`).
- **Floating + swap station destination:** `/deliver` requires `can_confirm_delivery === true`.
- **Firm trips:** `/pickup` validates `battery_ids` against pre-assigned items. `/deliver` validates completed returns exist per battery, but does not use the floating-only `can_confirm_delivery` guard. Progress endpoints are still recommended for UI consistency.

---

## App restart — which progress endpoint?

On movement screen load, use **`trip_status`** + **location type** (do not rely on locally stored lend/return IDs):

```
if trip_status === "accepted" && source.type === "swap_station"
  → GET source-pickup-progress

else if trip_status in ["picked_up", "in_transit"] && destination.type === "swap_station"
  → GET destination-delivery-progress

else
  → no progress endpoint; use trip resource + source-batteries / picked_up_batteries
```

---

## Counting rule (critical)

```javascript
// Wrong — counts unlock attempts including failures
target = localLendIds.length;

// Correct — always use server progress for swap-station IoT phases
target = progress.requested_count;        // pickup (or expected_count for delivery)
completed = progress.completed_count;
enableConfirm = progress.can_confirm_pickup // or can_confirm_delivery
```

On each lend/return poll, use the per-record flags:

| Poll field | Meaning |
|------------|---------|
| `is_terminal` | Stop polling this record |
| `counts_toward_pickup` | This completed **lend** counts toward pickup |
| `counts_toward_delivery` | This completed **return** counts toward delivery |

When `is_terminal === true`, refresh the relevant progress endpoint (do not increment local counters from failed records).

---

## Terminal lend/return statuses

| Status | App action |
|--------|------------|
| `pending` | Keep polling `GET lend-returns/{id}` |
| `completed` | Stop polling; refresh progress endpoint |
| `abandoned` | Stop polling; show retry; **do not count** toward total |
| `cancelled` | Stop polling; show retry; **do not count** toward total |

---

## Full flows

### 1. Floating trip — swap station → swap station

The most common IoT-heavy flow (e.g. incident trip 94).

#### Setup

1. `POST .../trips/{trip}/accept`
2. Load trip — confirm `type === "floating"`, `trip_status === "accepted"`

#### Source (pickup)

| Step | API | Body / notes |
|------|-----|--------------|
| 1 | `GET .../trips/{trip}/source-batteries` | Show available slots/batteries for rider selection |
| 2 | `POST .../trips/{trip}/unlock-source` | `{ "slot_ids": [365] }` — **required**; one or more slots per call |
| 3 | `GET .../lend-returns/{id}` | Poll each returned lend id until `is_terminal` |
| 4 | `GET .../trips/{trip}/source-pickup-progress` | After each terminal poll; on screen load |
| 5 | Enable **Confirm Pick** | When `can_confirm_pickup === true` |
| 6 | `POST .../trips/{trip}/pickup` | `{ "battery_ids": [...] }` from `completed_batteries[].battery_id` |

Repeat steps 2–4 for each slot until `remaining_count === 0`.

**Unlock guards (floating):**

- `slot_ids` required (422 if missing)
- Rejects if pickup already complete
- Rejects if slot already has a **pending** lend for this trip (poll existing lend instead of re-unlocking)
- Retry allowed after `abandoned` / `cancelled`

#### Destination (delivery)

| Step | API | Body / notes |
|------|-----|--------------|
| 1 | `GET .../trips/{trip}/destination-slots` | Empty slots at destination |
| 2 | `POST .../trips/{trip}/unlock-destination` | `{ "slot_id": 371, "battery_id": 1053 }` — one battery per call |
| 3 | `GET .../lend-returns/{id}` | Poll until `is_terminal` |
| 4 | `GET .../trips/{trip}/destination-delivery-progress` | After each terminal poll; on screen load |
| 5 | Enable **Confirm Delivery** | When `can_confirm_delivery === true` |
| 6 | `POST .../trips/{trip}/deliver` | `{ "fulfilled_count": N }` where `N === expected_count` |

Repeat steps 2–4 for each picked-up battery.

---

### 2. Firm trip — swap station → swap station

Batteries are known upfront (`pre_assigned_batteries`).

#### Source (pickup)

| Step | API | Body / notes |
|------|-----|--------------|
| 1 | `GET .../trips/{trip}/source-batteries` | Only pre-assigned batteries in slots |
| 2 | `POST .../trips/{trip}/unlock-source` | `{}` or omit `slot_ids` — backend derives slots from pre-assigned batteries |
| 3 | `GET .../lend-returns/{id}` | Poll each lend until terminal |
| 4 | `GET .../trips/{trip}/source-pickup-progress` | Optional but recommended for UI counters |
| 5 | `POST .../trips/{trip}/pickup` | `{ "battery_ids": [pre-assigned ids] }` — must match trip items |

**Difference from floating:** `/pickup` validates pre-assigned `battery_ids`, not `can_confirm_pickup`. Still use IoT unlock at swap stations before physical pickup.

#### Destination (delivery)

| Step | API | Body / notes |
|------|-----|--------------|
| 1 | `GET .../trips/{trip}/destination-slots` | Choose empty slots |
| 2 | `POST .../trips/{trip}/unlock-destination` | `{ "slot_id", "battery_id" }` per pre-assigned/picked-up battery |
| 3 | Poll + optional progress | Same as floating |
| 4 | `POST .../trips/{trip}/deliver` | `{ "fulfilled_count": N }` — backend resolves count from delivered items |

Each picked-up battery must have a **completed** return linked to the trip before deliver succeeds.

---

### 3. Floating trip — swap station → storage location

#### Source

Same as **§1 Source** (IoT unlock + progress + pickup).

#### Destination

No `unlock-destination`, no `destination-delivery-progress`, no `destination-slots`.

| Step | API | Body |
|------|-----|------|
| 1 | Rider places batteries at storage physically | — |
| 2 | `POST .../trips/{trip}/deliver` | `{ "fulfilled_count": N }` optional `battery_ids` (defaults to all picked-up items) |

---

### 4. Firm trip — swap station → storage location

#### Source

Same as **§2 Source** (unlock all pre-assigned slots, pickup with pre-assigned ids).

#### Destination

Direct deliver — no IoT return flow.

---

### 5. Floating trip — storage location → swap station

#### Source

No `unlock-source`, no `source-pickup-progress`.

| Step | API | Body |
|------|-----|------|
| 1 | `GET .../trips/{trip}/source-batteries` | Batteries at storage |
| 2 | `POST .../trips/{trip}/pickup` | `{ "battery_ids": [chosen ids] }` — count should match `requested_count` |

#### Destination

Same as **§1 Destination** (IoT return + progress + deliver).

---

### 6. Firm trip — storage location → swap station

#### Source

| Step | API | Body |
|------|-----|------|
| 1 | `GET .../trips/{trip}/source-batteries` | Pre-assigned batteries only |
| 2 | `POST .../trips/{trip}/pickup` | `{ "battery_ids": [pre-assigned ids] }` |

#### Destination

Same as **§2 Destination**.

---

### 7. Storage → storage (firm or floating)

No IoT unlock or progress endpoints.

| Phase | API |
|-------|-----|
| Pickup | `GET source-batteries` → `POST pickup` with appropriate `battery_ids` |
| Delivery | `POST deliver` with `fulfilled_count` |

---

### 8. Custodian depot destination

When destination is a custodian depot storage location:

| Step | API | Result |
|------|-----|--------|
| 1 | `POST .../trips/{trip}/deliver` | Sets `arrived_at`; trip may remain `in_transit` |
| 2 | Depot rep confirms separately | Out of field-executor app scope |

Batteries are **not** assigned to the depot until custodian confirmation.

---

## API reference

### Trip lifecycle

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/movement-requests/assigned` | Trips assigned to current user |
| `GET` | `/movement-requests/{id}` | Request detail with trips |
| `POST` | `/movement-requests/trips/{trip}/accept` | Accept trip |
| `POST` | `/movement-requests/trips/{trip}/decline` | Decline with reason |
| `POST` | `/movement-requests/trips/{trip}/cancel` | Cancel trip with reason |

### Source phase

| Method | Path | Body |
|--------|------|------|
| `GET` | `/movement-requests/trips/{trip}/source-batteries` | — |
| `POST` | `/movement-requests/trips/{trip}/unlock-source` | `{ "slot_ids": [int] }` — required for **floating**; optional/ignored for **firm** |
| `GET` | `/movement-requests/trips/{trip}/source-pickup-progress` | — |
| `POST` | `/movement-requests/trips/{trip}/pickup` | `{ "battery_ids": [int] }` |

### Destination phase

| Method | Path | Body |
|--------|------|------|
| `GET` | `/movement-requests/trips/{trip}/destination-slots` | — (swap station only) |
| `POST` | `/movement-requests/trips/{trip}/unlock-destination` | `{ "slot_id": int, "battery_id": int }` |
| `GET` | `/movement-requests/trips/{trip}/destination-delivery-progress` | — |
| `POST` | `/movement-requests/trips/{trip}/deliver` | `{ "fulfilled_count": int, "battery_ids"?: [int] }` |

### IoT polling

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/lend-returns/{id}` | Poll single lend/return status |

---

## Response shapes

### `GET source-pickup-progress`

```json
{
  "requested_count": 2,
  "completed_count": 1,
  "remaining_count": 1,
  "can_confirm_pickup": false,
  "completed_batteries": [
    {
      "battery_id": 1053,
      "physical_id": "BE84",
      "lend_id": 1002,
      "completed_at": "2026-07-22T10:15:00+00:00"
    }
  ],
  "active_lends": [
    {
      "lend_id": 1004,
      "battery_id": 1766,
      "physical_id": "BK72",
      "slot_id": 371,
      "status": "pending"
    }
  ],
  "failed_lends": [
    {
      "lend_id": 1001,
      "battery_id": 999,
      "physical_id": "BB42",
      "slot_id": 365,
      "status": "abandoned"
    }
  ]
}
```

### `GET destination-delivery-progress`

```json
{
  "expected_count": 2,
  "completed_count": 2,
  "remaining_count": 0,
  "can_confirm_delivery": true,
  "completed_batteries": [
    {
      "battery_id": 1053,
      "physical_id": "BE84",
      "return_id": 9277,
      "completed_at": "2026-07-22T11:00:00+00:00"
    }
  ],
  "active_returns": [],
  "failed_returns": []
}
```

### `GET lend-returns/{id}`

```json
{
  "id": 1002,
  "type": "lend",
  "status": "completed",
  "battery_id": 1053,
  "physical_id": "BE84",
  "slot_id": 365,
  "movement_trip_id": 94,
  "is_terminal": true,
  "counts_toward_pickup": true,
  "counts_toward_delivery": false
}
```

---

## Recommended polling loop

```javascript
async function pollLendReturn(lendReturnId) {
  while (true) {
    const { data } = await getLendReturn(lendReturnId);

    if (data.is_terminal) {
      await refreshProgressEndpoint(); // source or destination based on phase
      return data;
    }

    await sleep(POLL_INTERVAL_MS);
  }
}

async function refreshProgressEndpoint(trip) {
  if (trip.trip_status === 'accepted' && trip.source.type === 'swap_station') {
    return getSourcePickupProgress(trip.id);
  }

  if (['picked_up', 'in_transit'].includes(trip.trip_status)
      && trip.destination.type === 'swap_station') {
    return getDestinationDeliveryProgress(trip.id);
  }

  return null;
}
```

---

## Common errors (422)

| Code | When |
|------|------|
| `floating_trip_requires_slot_ids` | Floating unlock-source without `slot_ids` |
| `slot_unlock_already_pending` | Pending lend already exists on slot — poll it instead |
| `source_pickup_already_complete` | Floating unlock when enough lends already completed |
| `source_pickup_incomplete` | Floating pickup before enough completed lends |
| `destination_delivery_incomplete` | Floating deliver before enough completed returns |
| `slot_return_already_pending` | Pending return on slot — poll instead |
| `battery_not_from_completed_source_lends` | Pickup `battery_ids` don't match completed lends |

---

## Quick comparison: firm vs floating

| | Firm | Floating |
|---|------|----------|
| Batteries known upfront | Yes (`pre_assigned_batteries`) | No (only `requested_count`) |
| `unlock-source` `slot_ids` | Ignored — backend uses pre-assigned | **Required** |
| Trip items created | At booking | On `/pickup` |
| `/pickup` validation | `battery_ids` must match pre-assigned | Swap station: must match **completed lends** |
| `/deliver` validation (swap station) | Each battery needs completed return | Same + **`can_confirm_delivery` required** |
| Progress endpoints | Recommended for UI | **Required** for confirm buttons (swap station IoT) |

---

## Related docs

- [Battery movement issues & fixes](./battery-movement-issues-and-fixes.md) — incident analysis and backend rationale
