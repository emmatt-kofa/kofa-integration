# Motorbike Immobilization on Atlas — Frontend Guide

Atlas now shows whether a motorbike is immobilized, and the full history of every time it has
been. This is **read-only**: immobilizing, remobilizing and cancelling still happen from the
Partner app. Atlas only displays the result.

For how immobilization works end to end (the 24-hour countdown, the battery commands, timing),
see the Partner guide in `docs/motorbike-immobilization.md`.

## Where it shows up

| Endpoint | What's added |
|---|---|
| `GET /api/v1/atlas/inventory/motorbikes` | `immobilization_status` on each motorbike |
| `GET /api/v1/atlas/inventory/motorbikes/{motorbike}` | `immobilization_status` and `immobilizations` (the history) |
| `GET /api/v1/atlas/riders/{rider}` | `immobilization_status` on each motorbike under `accounts[].motorbikes` |

The history is only on the motorbike detail endpoint. The list and rider endpoints only carry the
current status.

## `immobilization_status`

```json
"immobilization_status": { "value": "immobilized", "label": "Immobilized" }
```

Same shape as the motorbike's `status` field. Show `label` as the text and drive styling off
`value`.

| `value` | `label` | What it means | Suggested badge |
|---|---|---|---|
| `active` | Active | The bike is usable. Covers bikes never immobilized, remobilized, cancelled, and bikes still counting down to immobilization. | None, or neutral |
| `request_sent` | Request Sent | The immobilize command was sent to the battery; waiting for it to acknowledge. | Amber, "Immobilizing…" |
| `request_received` | Request Received | The battery acknowledged; waiting for it to confirm it's actually throttled. | Amber, "Immobilizing…" |
| `immobilized` | Immobilized | The battery confirmed it's throttled. The bike is unusable. | Red, "Immobilized" |

A bike that has been initiated but is still in its 24-hour countdown reports `active`. Atlas can't
show "pending immobilization" from this field. Use the history's latest entry if you need that
(see "Working out a cycle's outcome" below).

## `immobilizations` (motorbike detail only)

A list of every immobilization cycle the bike has been through, **newest first**. A bike that has
never been immobilized returns `[]`.

```json
"immobilizations": [
  {
    "id": 9,
    "status": { "value": "immobilized", "label": "Immobilized" },
    "initiated_by": { "id": 2102, "name": "Abolaji Odukoya" },
    "rider": { "id": 929, "name": "Emmanuel Test Prod" },
    "initiated_at": "2026-10-16T08:00:00+00:00",
    "available_at": "2026-10-17T08:00:00+00:00",
    "sent_at": "2026-10-17T08:00:41+00:00",
    "received_at": "2026-10-17T08:00:53+00:00",
    "immobilized_at": "2026-10-17T08:01:40+00:00",
    "remobilization_sent_at": null,
    "remobilization_received_at": null,
    "remobilized_at": null,
    "cancellation_sent_at": null,
    "cancelled_at": null,
    "cancellation_unconfirmed_at": null,
    "retry_count": 0,
    "created_at": "2026-10-16T08:00:00+00:00"
  },
  {
    "id": 7,
    "status": { "value": "active", "label": "Active" },
    "initiated_by": { "id": 2102, "name": "Abolaji Odukoya" },
    "rider": { "id": 929, "name": "Emmanuel Test Prod" },
    "initiated_at": "2026-10-15T14:00:00+00:00",
    "available_at": "2026-10-16T14:00:00+00:00",
    "sent_at": null,
    "received_at": null,
    "immobilized_at": null,
    "remobilization_sent_at": null,
    "remobilization_received_at": null,
    "remobilized_at": null,
    "cancellation_sent_at": null,
    "cancelled_at": "2026-10-15T15:10:00+00:00",
    "cancellation_unconfirmed_at": null,
    "retry_count": 0,
    "created_at": "2026-10-15T14:00:00+00:00"
  },
  {
    "id": 5,
    "status": { "value": "active", "label": "Active" },
    "initiated_by": { "id": 2102, "name": "Abolaji Odukoya" },
    "rider": { "id": 929, "name": "Emmanuel Test Prod" },
    "initiated_at": "2026-10-15T11:23:13+00:00",
    "available_at": "2026-10-15T11:25:58+00:00",
    "sent_at": "2026-10-15T11:26:39+00:00",
    "received_at": "2026-10-15T11:26:51+00:00",
    "immobilized_at": "2026-10-15T11:27:40+00:00",
    "remobilization_sent_at": "2026-10-15T11:29:36+00:00",
    "remobilization_received_at": "2026-10-15T11:29:52+00:00",
    "remobilized_at": "2026-10-15T11:30:58+00:00",
    "cancellation_sent_at": null,
    "cancelled_at": null,
    "cancellation_unconfirmed_at": null,
    "retry_count": 0,
    "created_at": "2026-10-15T11:23:13+00:00"
  }
]
```

In this example, cycle 9 is the bike's current state (immobilized), cycle 7 was cancelled during
the countdown before anything was sent, and cycle 5 was immobilized and then remobilized.

### Fields

| Field | Meaning |
|---|---|
| `id` | The cycle's id. |
| `status` | Where the cycle got to on the immobilize side. Goes back to `active` once the cycle finishes, so **don't use it to label a finished cycle** — see below. |
| `initiated_by` | The partner user who started it. `null` for older cycles (see "Older cycles"). |
| `rider` | The bike's rider when the cycle started. `null` for older cycles. |
| `initiated_at` | When the partner started it. The countdown starts here. |
| `available_at` | When the countdown ends and the immobilize command fires automatically. |
| `sent_at` | The immobilize command was sent to the battery. |
| `received_at` | The battery acknowledged it. |
| `immobilized_at` | The battery confirmed it's throttled. This is when the bike actually became unusable. |
| `remobilization_sent_at` / `remobilization_received_at` / `remobilized_at` | The same three steps for remobilizing. `remobilized_at` set means the bike is usable again. |
| `cancellation_sent_at` | A cancellation was sent to the battery (only happens if the immobilize command had already gone out). |
| `cancelled_at` | The cycle was cancelled. If `sent_at` is `null`, it was cancelled during the countdown and the battery was never touched. |
| `cancellation_unconfirmed_at` | The battery never acknowledged the cancellation after about an hour. Ops may need to intervene. |
| `retry_count` | How many times a command had to be resent. Above 0 points to a battery that was slow or offline. |
| `created_at` | When the cycle was recorded. Use it as the cycle's date when `initiated_at` is `null`. |

All timestamps are ISO 8601 in UTC. Convert to local time for display.

### Working out a cycle's outcome

A cycle has **finished** once `remobilized_at` or `cancelled_at` is set. Read the timestamps,
most decisive first:

```ts
type Immobilization = {
  sent_at: string | null;
  immobilized_at: string | null;
  remobilization_sent_at: string | null;
  remobilized_at: string | null;
  cancellation_sent_at: string | null;
  cancelled_at: string | null;
};

function outcome(cycle: Immobilization): string {
  if (cycle.remobilized_at) return 'Remobilized';
  if (cycle.cancelled_at) return 'Cancelled';
  if (cycle.remobilization_sent_at) return 'Remobilizing';
  if (cycle.cancellation_sent_at) return 'Cancelling';
  if (cycle.immobilized_at) return 'Immobilized';
  if (cycle.sent_at) return 'Immobilizing';
  return 'Pending'; // initiated, counting down to available_at
}
```

Only the first entry (`immobilizations[0]`) can be in progress. Every other entry has finished.

### Showing a cycle as a timeline

For each cycle, list the steps that have a timestamp and skip the ones that are `null`:

1. Initiated — `initiated_at`, by `initiated_by.name`
2. Command sent — `sent_at`
3. Acknowledged — `received_at`
4. Immobilized — `immobilized_at`
5. Remobilization sent — `remobilization_sent_at`
6. Remobilization acknowledged — `remobilization_received_at`
7. Remobilized — `remobilized_at`
8. Cancellation sent — `cancellation_sent_at`
9. Cancelled — `cancelled_at`

If `cancellation_unconfirmed_at` is set, show a warning on that cycle.

## Older cycles

Cycles from before the history was introduced were copied over from what the motorbike record
held at the time, which didn't include who started a cycle or who was riding. On those:

- `initiated_by`, `rider` and `initiated_at` are `null`
- some step timestamps may be missing

Show them with "—" for the missing people and `created_at` as the date. Don't hide them.

## Checklist

- [ ] Status badge on the motorbike list, motorbike detail header and rider detail page, from `immobilization_status`
- [ ] History timeline on the motorbike detail page, from `immobilizations`, in the order returned
- [ ] Outcome for each cycle worked out from the timestamps, not from `status`
- [ ] Empty state when `immobilizations` is `[]`
- [ ] `null` `initiated_by` / `rider` / `initiated_at` handled
- [ ] Warning when `cancellation_unconfirmed_at` is set
- [ ] Timestamps shown in local time
