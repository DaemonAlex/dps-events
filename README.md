<p align="center"><img src=".github/dps-logo.svg" alt="DelPerroSands" width="360"></p>

# dps-events

**Playtime reward events for DelPerroSands.** Staff create server-wide events; players earn a reward for staying connected and playing for a set amount of time. Progress is tracked live in-game and the reward is granted automatically on completion.

## Status

Unfinished: planning only. This repo holds the plan below and no code. Nothing from it runs on
DelPerroSands.

## What it is for

Staff want a simple way to run an event such as "play two hours today and get a reward". Players
get a clear reason to stay online. The idea comes from the BVS Events Creator feature set, to be
built for the DelPerroSands Qbox stack.

## On its own, and with other scripts

**On its own.** There is nothing to run yet. The plan needs `qbx_core`, `oxmysql` and `ox_lib`.
A player would see a small panel with their session time, the event name, a progress bar and the
reward. The reward would arrive by itself when the time is reached.

**With ox_inventory (planned).** Item rewards would be given through `ox_inventory`.

**With a paid dealership script (jg-dealerships) (planned).** Vehicle rewards would use the same
ownership path that jg-dealerships uses.

None of these links exist in code yet.

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
| **Vehicle** | temporary (removed next restart) **or** permanent (added to owned vehicles) | qbx vehicle registry + `player_vehicles` |
| **VIP Coins** | added to a permanent account balance | needs a coin account — see Open questions |
| **Money** | configurable amount to Cash / Bank / Black Money | `qbx_core` money functions |
| **Item** | any inventory item + quantity | `exports.ox_inventory:AddItem` |

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

## Recent changes

- **12 November 2026.** README: status, what it is for, how it works alone and with paid scripts, other DPS scripts.
- **27 August 2026.** Plan written. README only.

## Author

Made by [DaemonAlex](https://github.com/DaemonAlex) for DelPerroSands.

Inspired by the BVS Events Creator feature set. Built for Qbox, not bought.

## Other DPS scripts

All of these are made for DelPerroSands and run together on one Qbox server. Each one works on
its own. The ones marked "pairs with" do more when both are running.

| Script | What it does |
|---|---|
| [dps-inventory](https://github.com/DaemonAlex/dps-inventory) | The inventory: ox_inventory with equipment slots, backpacks, armour plates and a self status panel, in the DelPerroSands colours. |
| [dps-medical](https://github.com/DaemonAlex/dps-medical) | Patient records, illness and diagnostic tests. A companion to wasabi_ambulance. Pairs with dps-inventory: symptoms and diagnoses show on the status panel. |
| [dps-insurance](https://github.com/DaemonAlex/dps-insurance) | Ties m-Insurance into the MDT, impound, hospital and medical chart, and raises premiums for incidents other scripts report. Pairs with dps-medical and dps-fuel. |
| [dps-studio](https://github.com/DaemonAlex/dps-studio) | Build rooms, doors and portals in game from one panel (`/studio`), with owners and managers so staff do not have to support them. |
| [dps-shops](https://github.com/DaemonAlex/dps-shops) | Shop clerks who greet, react to robberies, talk back through dps-ainpcs and remember trouble, plus player-owned stores with delivery runs. Pairs with dps-ainpcs. |
| [dps-fleet](https://github.com/DaemonAlex/dps-fleet) | Vehicle browser and workshop on one panel: extras, liveries and mods for emergency vehicles. |
| [dps-parking](https://github.com/DaemonAlex/dps-parking) | Parking with valet, impound, violations and reserved spots. |
| [dps-vehiclepersistence](https://github.com/DaemonAlex/dps-vehiclepersistence) | Vehicles stay where they were parked. Pairs with dps-parking. |
| [dps-towjob](https://github.com/DaemonAlex/dps-towjob) | Queue-based tow job with jg-mechanic integration. |
| [dps-services](https://github.com/DaemonAlex/dps-services) | Request a tow and follow it live on the lb-phone and lb-tablet. Pairs with dps-towjob. |
| [dps-autoshops](https://github.com/DaemonAlex/dps-autoshops) | Self-service repair shops when no mechanic is on duty. |
| [dps-cityworker](https://github.com/DaemonAlex/dps-cityworker) | City infrastructure job and career. |
| [dps-ainpcs](https://github.com/DaemonAlex/dps-ainpcs) | NPCs you can talk to, with trust, quests and intel. Other scripts can make their own NPCs talk (dps-shops clerks). |
| [dps-badpeds](https://github.com/DaemonAlex/dps-badpeds) | Police interaction with NPCs: stops, arrests, intel and jail records. |
| [dps-gangwars](https://github.com/DaemonAlex/dps-gangwars) | Ambient gang NPCs, territory population and war reinforcements. |
| [dps-rotations](https://github.com/DaemonAlex/dps-rotations) | Moves dealers, harvests and other spots on a fixed schedule. |
| [dps-trains](https://github.com/DaemonAlex/dps-trains) | The train system. Pairs with dps-trains-stock, dps-traintools and dps-transitapp. |
| [dps-transitapp](https://github.com/DaemonAlex/dps-transitapp) | Live train arrival boards on the lb-phone. |
| [dps-faceplant](https://github.com/DaemonAlex/dps-faceplant) | City help, reports and staff tools on the lb-phone and lb-tablet. |
| [dps-chat](https://github.com/DaemonAlex/dps-chat) | A command box for voice-only servers. No chat feed. |
| [dps-filmmode](https://github.com/DaemonAlex/dps-filmmode) | One key hides every HUD layer and gives a smooth free camera for recording. |
| [dps-fuel](https://github.com/DaemonAlex/dps-fuel) | Gas station drive-offs for the ox_fuel nozzle: ripped hose, pump fire, station camera and a police call, by odds. Pairs with dps-insurance. |
| [dps-voice](https://github.com/DaemonAlex/dps-voice) | Spoken lines for NPCs, station announcements, dispatch and shops, from local voice models first. Pairs with dps-ainpcs. |
| [dps-bags](https://github.com/DaemonAlex/dps-bags) | Real bags for ox_inventory: the bag you carry shows on your character. |
| [dps-items](https://github.com/DaemonAlex/dps-items) | Item rules for ox_inventory: locked items, glovebox seats and an exploit watch. |
| [dps-furniture](https://github.com/DaemonAlex/dps-furniture) | Puts streamed furniture packs into the qs-housing furniture shop. |
| [dps-shellbrowser](https://github.com/DaemonAlex/dps-shellbrowser) | Staff tool to look at every qs-housing shell and set its door. |
| [dps-signs](https://github.com/DaemonAlex/dps-signs) | Rotates city messages across the SmartSigns road boards. |
| [dps-stationclear](https://github.com/DaemonAlex/dps-stationclear) | Hides map props that are in the way and keeps stray game peds off chosen spots. |
| [dps-citymap](https://github.com/DaemonAlex/dps-citymap) | A ready-made map of the whole city, cut into tiles for in-game screens. |
| [dps-cayo](https://github.com/DaemonAlex/dps-cayo) | Switches on the game's own Cayo Perico island, and nothing else. |
| [dps-aviation](https://github.com/DaemonAlex/dps-aviation) | Flight behaviour: aircraft idle instead of cutting out, sane taxi speed, a real start sequence. |
| [dps-airlines](https://github.com/DaemonAlex/dps-airlines) | Airlines job: passenger flights, cargo, charters, flight school and maintenance. |
| [dps-traintools](https://github.com/DaemonAlex/dps-traintools) | Rider tools for trains: board, sit, get off at a stop, ambient passengers. Pairs with dps-trains. |
| [dps-trains-stock](https://github.com/DaemonAlex/dps-trains-stock) | Train models and carriage sets for the train system. No scripts. Pairs with dps-trains. |
| [dps-traintracks](https://github.com/DaemonAlex/dps-traintracks) | Extra train track data. No scripts. Pairs with dps-trains. |
| [dps-crossings](https://github.com/DaemonAlex/dps-crossings) | Level crossings that close when a train comes. Pairs with dps-trains. |
| [dps-transit](https://github.com/DaemonAlex/dps-transit) | Passenger rail planning and scheduling (older system; dps-trains runs today). |
| [dps-maritime](https://github.com/DaemonAlex/dps-maritime) | Dock work and boat deliveries with progression. |
| [dps-maritime-maps](https://github.com/DaemonAlex/dps-maritime-maps) | Map files for the Jetsam docks, the map half of dps-maritime. |
| [dps-proracing](https://github.com/DaemonAlex/dps-proracing) | Circuit racing with lobbies, an AI practice grid and lap times. |
| [dps-carmenu](https://github.com/DaemonAlex/dps-carmenu) | Staff vehicle browser and spawner over the live Qbox vehicle list. |
| [dps-clothingbrowser](https://github.com/DaemonAlex/dps-clothingbrowser) | Staff tool to browse, identify and export clothing and uniforms. |
| [dps-lootpeds](https://github.com/DaemonAlex/dps-lootpeds) | Loot dead NPCs, with loot tables per model. |
| [dps-hookers](https://github.com/DaemonAlex/dps-hookers) | Adults-only RP (18+), adapted from MH Hookers by MaDHouSe. |
| [dps-weatherbridge](https://github.com/DaemonAlex/dps-weatherbridge) | Lets qb-weathersync and Renewed-Weathersync scripts work with night_natural_disasters. |
| [dps-fixes](https://github.com/DaemonAlex/dps-fixes) | Small server fixes kept in one resource. |
| [dps-bucketfix](https://github.com/DaemonAlex/dps-bucketfix) | Puts players back in the main world after spawning. |
| [dps-unfocus](https://github.com/DaemonAlex/dps-unfocus) | One key that frees a player stuck in a menu. |
| [dps-welcomecam](https://github.com/DaemonAlex/dps-welcomecam) | Aims the first camera a new player sees. |
| [dps-handlingdump](https://github.com/DaemonAlex/dps-handlingdump) | Developer tool: saves the stock handling of every vanilla vehicle. |
| [dps-weaponsounds](https://github.com/DaemonAlex/dps-weaponsounds) | Custom weapon sounds (planning; no code yet). |
