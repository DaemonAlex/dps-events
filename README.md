# dps-events

**Playtime reward events for DelPerroSands.** Staff create server-wide events; players earn a reward for staying connected and playing for a set amount of time. Progress is tracked live in-game and the reward is granted automatically on completion.

> Status: **planning** — this repo currently contains the plan only, no code yet.
> Inspired by the BVS Events Creator feature set, rebuilt natively on the DPS/Qbox stack.

---

## Goal

Give staff a simple tool to run engagement events ("play 2 hours today, get X"), and give players a clear reason to stay online — feeding the longer-term [staff portal](#relationship-to-other-dps-systems) vision.

## Core concept

1. A staff member creates an **event**: a required playtime + a reward.
2. The event is announced server-wide and becomes the single active event.
3. Every connected player accumulates **session playtime** while the event runs.
4. On reaching the required time, the player is **automatically granted the reward**.
5. Staff can replace or cancel the active event at any time.

Only **one event is active at a time** (matches the reference design; keeps progress unambiguous).

---

## Features

### Playtime tracking
- Server-authoritative per-player session timer (never trust the client clock).
- The "same session" rule: progress accumulates only while continuously connected. **Disconnect resets session progress** for that event (design decision — see [Open questions](#open-questions)).
- Tick model: event-driven accrual (store `join_time`, compute elapsed) rather than a per-second loop — no polling ([[dps-no-polling-loops]]).

### Reward types
| Type | Behaviour | Backend |
|---|---|---|
| 🚗 **Vehicle** | temporary (removed next restart) **or** permanent (added to owned vehicles) | qbx vehicle registry + `player_vehicles` |
| 👑 **VIP Coins** | added to a permanent account balance | needs a coin account — see Open questions |
| 💵 **Money** | configurable amount to Cash / Bank / Black Money | `qbx_core` money functions |
| 📦 **Item** | any inventory item + quantity | `exports.ox_inventory:AddItem` |

### Player progress UI
- A "Session Playtime" panel: current session time, connection status, active event name, required time, progress bar, and the reward on offer.
- Updates live while connected. Small NUI (pattern: `dps-transitapp`).

### Global announcements
- Auto-announce on event start; optional custom staff message.
- Uses ox_lib notify / chat broadcast.

### Staff event management
- Create / replace / cancel the active event.
- ace-gated to staff (`group.admin` or a dedicated `dps.events` ace).
- Reward configured at creation time (type + params).

---

## Architecture (planned)

```
dps-events/
  fxmanifest.lua
  shared/
    config.lua          -- reward defaults, ace group, tick granularity, AFK policy
  server/
    main.lua            -- event lifecycle, session accrual, reward grant
    rewards.lua         -- one handler per reward type (vehicle/coins/money/item)
    db.lua              -- oxmysql access
  client/
    main.lua            -- session UI trigger, progress events
    nui.lua             -- progress panel bridge
  web/                  -- progress panel (built NUI)
  sql/
    schema.sql
```

### Data model (draft)
```sql
-- the current/active event (one row conceptually; history kept for audit)
CREATE TABLE dps_events (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(80) NOT NULL,
  required_seconds INT NOT NULL,
  reward_type ENUM('vehicle','coins','money','item') NOT NULL,
  reward_data JSON NOT NULL,          -- {model, permanent} | {amount} | {account, amount} | {item, qty}
  status ENUM('active','cancelled','ended') NOT NULL DEFAULT 'active',
  created_by VARCHAR(64) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- per-player progress for the active event
CREATE TABLE dps_event_progress (
  event_id INT NOT NULL,
  citizenid VARCHAR(64) NOT NULL,
  accrued_seconds INT NOT NULL DEFAULT 0,
  session_start BIGINT,               -- epoch of current connection segment (NULL when offline)
  rewarded TINYINT NOT NULL DEFAULT 0,
  PRIMARY KEY (event_id, citizenid)
);
```

### Reward grant flow
1. On each accrual tick (join/leave, and a coarse safety interval), server recomputes `accrued_seconds`.
2. When `accrued_seconds >= required_seconds` and `rewarded = 0`: run the reward handler for `reward_type`, set `rewarded = 1`, notify the player.
3. Grants are **idempotent** and server-side only — a player can never self-report completion.

---

## Dependencies
- `qbx_core` (identity, money, vehicles)
- `oxmysql` (persistence)
- `ox_inventory` (item rewards)
- `ox_lib` (menus, notify, UI)

## Relationship to other DPS systems
- **Staff portal (future):** this is a natural first staff-run engagement tool; event creation/reporting could later surface in the web portal.
- **Vehicle rewards:** reuse the ownership/registry path already used by jg-dealerships.
- **No polling:** accrual is event-driven per the standing rule.

---

## Open questions (decide before build)
1. **VIP Coins** — do we have (or want) a coin/VIP currency? If not, drop that reward type or map it to an existing account for v1.
2. **AFK farming** — track *connection* time (simple, farmable) or *active* time (movement/input gating)? Reference design uses connection time; DPS may want active.
3. **Disconnect policy** — hard reset session progress on disconnect (reference behaviour), or allow a short grace reconnect window?
4. **Multiple concurrent events** — reference allows one; keep that for v1?
5. **Temp vehicle cleanup** — "removed on next restart" — confirm the restart hook and owner mapping.

## Roadmap
- **v0 (this repo):** plan + schema.
- **v1:** one active event, money + item rewards, live progress UI, staff create/cancel.
- **v2:** vehicle + VIP-coin rewards, replace-active-event, custom announcements.
- **v3:** active-playtime (anti-AFK), event history/reporting, staff-portal hooks.

---

*DelPerroSands · built native on Qbox, not purchased. Plan authored 2026-08-27.*
