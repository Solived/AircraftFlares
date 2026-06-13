# Aircraft Countermeasure Flares — System Documentation

## Overview

`Aircraft Countermeasure Flares` is a standalone FiveM resource that adds a fully server-authoritative countermeasure flare system to any aircraft. It requires no framework (no ESX, QBCore, or vRP) and has no external dependencies beyond standard FiveM natives. Every sensitive action — ammo consumption, cooldowns, jam rolls, and rearming — is validated and processed on the server. The client handles rendering, visual effects, audio, and key input only.

The system supports unlimited aircraft with:
- Per-aircraft configuration
- Dual ejection banks (left and right)
- Four fire modes
- Damage-based jamming
- Zone-gated rearming with a progress timer
- Two HUD theme designs
- Three flare visual variants (default red, glowing gold, white IR)
and a public export API for missile guidance integration.

---

## Resource Structure

```
Solived-AircraftFlares/
├── fxmanifest.lua   — resource manifest
├── config.lua       — all configuration + two shared helper functions
├── client/
    └── main.lua       — rendering, effects, audio, input, key bindings
├── server/
    └── server.lua       — authority: state, ammo, cooldowns, jam, rearm
```

`config.lua` is declared as a **shared script**, meaning both the client and server load it. The two helper functions it exports (`GetAircraftConfig` and `IsAllowedSeat`) are used on both sides so logic stays consistent without duplication.

---

## Architecture & Authority Model

The server owns all state. The client never deducts its own ammo, starts its own cooldown, or decides whether a deploy is valid. The flow for every deploy is:

```
Player presses key
  → client canDeploy() [local pre-filter: seat, engine, altitude, speed]
    → TriggerServerEvent('Solived-AircraftFlares:deployRequest')
      → server validates occupant, seat, aircraft, cooldown, jam roll
        → deducts ammo, sets cooldown
          → TriggerClientEvent('Solived-AircraftFlares:doDeployOwner', src)     ← owner fires real projectile + replica
          → TriggerClientEvent('Solived-AircraftFlares:doDeployReplica', -1)    ← all others see replica only
          → TriggerClientEvent('Solived-AircraftFlares:updateHud', src)         ← HUD synced from server truth
```

The client-side `canDeploy()` check exists purely to avoid unnecessary network events for obvious failures (wrong aircraft, on the ground, engine off). It is not trusted for authority. All real decisions happen server-side.

---

## Configuration Reference

### General

```lua
Config.Debug = false
```
When `true`, prints tagged debug lines to the server/client console at key decision points (deploy, rearm, jam, mode change).

```lua
Config.CommandName   = 'flares'
Config.RefillCommand = 'refillflares'
Config.DefaultKey    = 'Y'
```
`CommandName` drives two things simultaneously: the `/flares <mode>` chat command for bank selection, and the hold-key binding (`+flares` / `-flares`). `DefaultKey` is the default keyboard bind registered via `RegisterKeyMapping` — players can rebind it freely in GTA settings. `RefillCommand` registers the `/refillflares` chat command for manual rearming.

---

### Seat Restriction

```lua
Config.SeatRestriction = {
    enabled   = true,
    pilotOnly = false,
    allowedSeats = {
        [-1] = true,
        [0]  = true
    }
}
```

Controls which seats may activate the flare system and trigger rearming.

| Field | Type | Behaviour |
|---|---|---|
| `enabled` | boolean | `false` = any occupied seat may use the system |
| `pilotOnly` | boolean | `true` = only seat index `-1` (the pilot) is permitted |
| `allowedSeats` | table | Used when `pilotOnly` is `false`. Keys are GTA seat indices: `-1` pilot, `0` co-pilot, `1`–`N` passengers |

Seat validation runs on **both** client (for local pre-filter) and server (for authority).

---

### Flight Checks

```lua
Config.RequireEngineOn    = true
Config.BlockWhenDestroyed = true
Config.AllowOnGround      = false

Config.Checks = {
    altitudeEnabled = true,
    speedEnabled    = false
}

Config.MinimumAltitude = 7.5   -- metres above ground
Config.MinimumSpeed    = 12.0  -- m/s (~43 km/h)
```

These are client-side pre-filters applied before a deploy request is sent to the server.

- `RequireEngineOn` — blocks deploy if `GetIsVehicleEngineRunning` returns false.
- `BlockWhenDestroyed` — blocks deploy if entity health is at or below zero.
- `AllowOnGround` — when `false`, blocks deploy if `IsVehicleOnAllWheels` returns true, regardless of altitude. Prevents flares while taxiing.
- `altitudeEnabled` — when `true`, measures vertical clearance from the ground via `GetGroundZFor_3dCoord` and blocks if below `MinimumAltitude`.
- `speedEnabled` — when `true`, blocks deploy if the aircraft's speed (m/s) is below `MinimumSpeed`. Useful if you want to prevent flares at taxi speeds only.

Both altitude and speed checks can be independently toggled without affecting each other.

---

### Deploy Behaviour

```lua
Config.Deploy = {
    holdRateMs           = 75,
    allowHoldRepeat      = true,
    holdRepeatIntervalMs = 120
}
```

Controls how the hold-key input translates to deploy events.

- `holdRateMs` — how often (ms) the hold loop wakes up to check if the key is still held. Lower = more responsive feel; higher = lighter CPU.
- `allowHoldRepeat` — when `true`, holding the key continuously fires bursts at the cooldown rate. When `false`, each key press fires exactly one burst regardless of how long the key is held (tap-only mode).
- `holdRepeatIntervalMs` — minimum gap between consecutive hold-triggered deploys. Acts as a local throttle on top of the server cooldown to prevent event spam.

---

### Damage & Jamming

```lua
Config.Damage = {
    enabled                  = true,
    jamHealthThreshold       = 650.0,
    jamChance                = 0.28,
    severeJamHealthThreshold = 350.0,
    severeJamChance          = 0.55,
    jamCooldownMultiplier    = 0.5
}
```

When `enabled` is `true`, the server evaluates vehicle health on every deploy request and may randomly jam the dispenser.

GTA vehicle health runs from **0 (destroyed)** to **1000 (full)**.

| Condition | Jam probability |
|---|---|
| Health > 650 | 0% — system fully operational |
| Health 351–650 | 28% (`jamChance`) per deploy attempt |
| Health ≤ 350 | 55% (`severeJamChance`) per deploy attempt |

When a jam occurs, no ammo is consumed. The dispenser enters a short lockout period equal to `cfg.cooldownMs × jamCooldownMultiplier`. The HUD displays `JAMMED` in red during this window. The player must wait out the lockout before the next attempt. Setting `enabled = false` disables all jam rolls entirely.

---

### Rearm / Refill

```lua
Config.Rearm = {
    useZones       = true,
    durationMs     = 7000,
    requireStopped = true,
    maxSpeed       = 2.0,
    adminOnly      = false,
    adminAce       = 'group.admin',
    zones = {
        { label = 'LSIA Hangar',  coords = vec3(-979.0, -2995.0, 13.9), radius = 35.0 },
        { label = 'Fort Zancudo', coords = vec3(-2133.0, 3254.0, 32.8), radius = 50.0 },
        { label = 'Carrier Deck', coords = vec3(3082.0, -4719.0, 15.0), radius = 65.0 }
    }
}
```

- `useZones` — when `false`, `/refillflares` works anywhere with no location requirement. When `true`, the player must be inside a defined zone radius.
- `durationMs` — rearming is not instant. The player must remain in the zone for this many milliseconds. A countdown is drawn on screen during the process. Controls that would let the player bail out are suppressed during this window.
- `requireStopped` / `maxSpeed` — when `requireStopped` is `true`, the aircraft must be moving slower than `maxSpeed` m/s to begin rearming. Prevents rearming mid-flight by flying through a zone.
- `adminOnly` / `adminAce` — when `adminOnly` is `true`, the `/refillflares` server command is gated behind a FiveM ACE permission check. Any ace identifier works (e.g. `group.admin`, `command.refillflares`). When `false`, any player in an allowed seat may rearm.
- `zones` — unlimited zone entries. Each needs a `label` (for your reference only, not shown to players), `coords` (vec3), and `radius` (metres). Add, remove, or reposition zones freely.

---

### Audio

```lua
Config.Audio = {
    useReleaseBank = true,
    bank           = 'DLC_SM_Countermeasures_Sounds',
    releaseSound   = 'flares_released',
    releaseSet     = 'DLC_SM_Countermeasures_Sounds',
    useCockpitTone = true,
    cockpitSound   = 'CONFIRM_BEEP',
    cockpitSet     = 'HUD_MINI_GAME_SOUNDSET'
}
```

- `useReleaseBank` / `bank` — when `true`, the DLC audio bank is requested before playing the release sound and released after the burst completes. Set to `false` if the bank is already loaded globally by another resource.
- `releaseSound` / `releaseSet` — the 3D positional sound played from the aircraft entity on deploy. Heard by nearby players.
- `useCockpitTone` / `cockpitSound` / `cockpitSet` — a 2D frontend beep played only on the deploying player's client as a tactile confirmation. Set `useCockpitTone = false` to remove it.

---

### Flare Projectile & Visual Replica

```lua
Config.Flare = {
    projectileSpeed  = -1.0,
    burstIntervalMs  = 120,
    hotDurationMs    = 2600,
    rearConeDegrees  = 75.0,

    visualReplica    = true,
    replicaModel     = `w_am_flare`,
    replicaLifeMs    = 6000,
    replicaFadeMs    = 3800,
    replicaScale     = 1.0,
    replicaJitter    = 0.3,
    replicaEjectSpeed = 7.5,
    replicaDropSpeed  = 1.8,

    defaultVariant   = 'gold',
    variants         = { ... }
}
```

**Projectile:**
- `projectileSpeed` — velocity passed to `ShootSingleBulletBetweenCoords`. `-1.0` uses the game's default for the weapon. Increase for faster-travelling flares.
- `burstIntervalMs` — delay in milliseconds between each flare fired within a single burst. 120ms gives a rapid staggered spread rather than all flares spawning simultaneously.

**Hot window:**
- `hotDurationMs` — how long (ms) the vehicle remains "hot" after a deploy, as tracked by the `IsVehicleCountermeasureHot` export. Each flare in the burst resets this window, so a long burst extends total hot time.
- `rearConeDegrees` — the angular half-cone behind the aircraft that defines the "rear sector." A missile approaching from within this cone behind the aircraft receives full hotness (1.0). A missile approaching from the front receives reduced hotness (0.35). 75° means anything within 75° of straight behind is considered rear.

**Visual replica:**
The replica is a prop object (`w_am_flare` by default) spawned at the ejection point with physics applied, giving a visible tumbling flare trail. It does not deal damage and has collision disabled.

- `visualReplica` — set `false` to skip all prop spawning (projectile still fires).
- `replicaModel` — any valid GTA prop model hash. The model is streamed at runtime if not already loaded.
- `replicaLifeMs` — total lifespan of the prop in milliseconds.
- `replicaFadeMs` — the prop begins fading out this many milliseconds before death. Set to `0` to disable fading (instant deletion).
- `replicaScale` — scale multiplier for the spawned prop. `1.0` = default game size.
- `replicaJitter` — random lateral velocity variance applied to each flare (m/s). Creates a spread pattern rather than a straight line. `0.0` = no spread.
- `replicaEjectSpeed` — rearward velocity applied to the replica relative to the aircraft's direction of travel (m/s). Higher values push flares further behind faster.
- `replicaDropSpeed` — downward velocity component applied to the replica (m/s). Controls how quickly flares fall away below the aircraft.

**Variants:**
Three variants are included. Any aircraft entry can specify `variant = 'name'` to override the default.

| Variant | Appearance | Glow |
|---|---|---|
| `red` | Standard GTA flare (no glow) | No |
| `gold` | Warm amber military countermeasure | Yes — 14m range, 8.0 intensity |
| `white` | Cool white IR/phosphorus decoy | Yes — 18m range, 10.0 intensity |

Each variant supports:
- `weapon` — the weapon hash used to fire the projectile
- `glow` — whether to draw a dynamic light from the replica prop each frame
- `glowColor` — RGB table `{ r, g, b }`
- `glowRange` — light radius in metres
- `glowIntensity` — light brightness multiplier
- `glowTimeMs` — how long (ms) after spawn the glow remains active
- `tintEnabled` / `tintIndex` — apply a weapon tint to the flare gun on the owner ped

Custom variants can be added freely by adding new entries to the `variants` table and referencing them by name.

---

### HUD

```lua
Config.HUD = {
    enabled = true,
    design  = 'tactical',
    Designs = { tactical = { ... }, compact = { ... } }
}
```

Set `design` to `'tactical'` or `'compact'` to switch the active layout. Both designs share the same colour key system.

**Tactical design** (full panel):
- Header row: system label left, live status right (`READY` / `CD 3.2s` / `JAMMED`)
- Dual ammo count: `L 024` and `R 024` with individual progress bars per bank
- Footer row: current fire mode left, total ammo right

**Compact design** (slim widget):
- Left accent stripe instead of top bar
- Single combined ammo count and progress bar
- Mode and per-bank breakdown on a small footer line

**Position and sizing** (per design):
- `x` / `y` — screen-space anchor in normalised coordinates (0.0–1.0). `0.885, 0.805` places the panel near the lower right.
- `scale` — uniform multiplier applied to all width, height, and text sizing values. `1.2` makes the panel 20% larger.
- `width` / `height` — base panel dimensions before scale is applied.

**Colour keys** (RGBA arrays, all values 0–255):
| Key | Used for |
|---|---|
| `accentColor` | Title text, top/side stripe, progress bar fill |
| `textColor` | Ammo count values, total count |
| `mutedColor` | Mode label, divider line, compact footer |
| `readyColor` | Status text when cooldown is 0 |
| `cooldownColor` | Status text during active cooldown |
| `jamColor` | Status text when dispenser is jammed |
| `backgroundColor` | Main panel fill |
| `panelColor` | (tactical only) Inner panel layer |
| `barBackgroundColor` | Empty portion of progress bars |

---

### Aircraft Profiles

Profiles define the physical ejection point offsets and serve as defaults for ammo, cooldown, and burst size.

```lua
Config.DefaultProfileAmmo     = { fighter = 300, heli = 24, heavy = 48 }
Config.DefaultProfileCooldown = { fighter = 200, heli = 4600, heavy = 5600 }
Config.DefaultProfileBurst    = { fighter = 2,   heli = 2,    heavy = 4   }
```

Three profiles ship by default: `fighter`, `heli`, and `heavy`. You can add new profile names to all three default tables and to `Config.Profiles` and reference them from any aircraft entry.

```lua
Config.Profiles = {
    fighter = {
        offsets = {
            left  = { vec3(-1.2, -4.2, -0.20), vec3(-1.9, -4.6, -0.15) },
            right = { vec3( 1.2, -4.2, -0.20), vec3( 1.9, -4.6, -0.15) }
        }
    },
    ...
}
```

Each offset is a `vec3(right, forward, up)` in the aircraft's local space. Negative forward values are behind the aircraft's origin; negative up values are below. Each bank can carry as many offset points as you like. During a burst, flares cycle through the offset list, so two offsets create a front-to-rear spread pattern within the bank.

**Tuning offsets for a custom aircraft:** spawn the aircraft, use a test resource to draw world markers at `GetOffsetFromEntityInWorldCoords(vehicle, x, y, z)`, and adjust `x`/`y`/`z` until the markers sit on the physical dispenser pods.

---

### Allowed Aircraft

```lua
Config.AllowedAircraft = {
    f35a  = { profile = 'fighter' },
    ah64e = { profile = 'heli', ammo = 32, cooldownMs = 4000 },
    ac130u = { profile = 'heavy', ammo = 64, burstSize = 4 },
    ...
}
```

The key must exactly match the GTA vehicle model name (lowercase). Any aircraft not listed here will be silently ignored — the system will not activate for it.

Per-aircraft overrides (all optional, all fall back to profile defaults if omitted):

| Field | Type | Description |
|---|---|---|
| `profile` | string | Which profile's offsets and defaults to use |
| `ammo` | number | Total flare count. Split evenly between left and right banks |
| `cooldownMs` | number | Milliseconds between bursts for this specific aircraft |
| `burstSize` | number | Flares fired per activation for this aircraft |
| `variant` | string | Visual variant override (`'red'`, `'gold'`, `'white'`, or custom) |

Ammo is always split evenly. If an odd number is given, the right bank carries the extra flare. For example `ammo = 33` gives left 16, right 17.

---

### Bank Fire Modes

Players change modes with `/flares <mode>`. The mode is stored server-side per vehicle (not per player), so all crew see the same mode on the HUD.

| Mode | Behaviour |
|---|---|
| `both` | Both banks fire simultaneously each burst (default) |
| `left` | Only the left bank fires |
| `right` | Only the right bank fires |
| `alternate` | Banks alternate each burst: L, R, L, R... |

`alternate` mode is tracked server-side via `state.alternateNext` so the sequence survives across multiple crew members and server restarts within the session.

---

## Export API

Other resources can query countermeasure state for missile guidance, lock-on defeat, or evasion logic:

```lua
local hot, hotness = exports.Solived-AircraftFlares:IsVehicleCountermeasureHot(vehicle, missileCoords, extraBurst)
```

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `vehicle` | number | The vehicle entity handle to check |
| `missileCoords` | vec3 or nil | World position of the incoming missile. Pass `nil` to skip directional weighting |
| `extraBurst` | boolean or nil | Pass `true` to apply a +0.15 bonus to hotness (e.g. if a second burst was fired very recently) |

**Returns:**

| Return | Type | Description |
|---|---|---|
| `hot` | boolean | `true` if flares are currently active on this vehicle |
| `hotness` | number | Effectiveness score 0.0–1.0 |

**Hotness scoring:**
- `1.0` — missile is approaching from the rear sector (within `rearConeDegrees` of directly behind). Full deflection.
- `0.35` — missile is approaching from the forward hemisphere. Flares are visible but less effective due to geometry.
- `+0.15` bonus — added when `extraBurst` is `true`, capped at `1.0`.

**Example integration (missile guidance resource):**
```lua
local function shouldMissileDeflect(missile, target)
    local missilePos = GetEntityCoords(missile)
    local hot, hotness = exports.Solived-AircraftFlares:IsVehicleCountermeasureHot(target, missilePos, false)
    if hot and math.random() < hotness then
        -- deflect missile
        return true
    end
    return false
end
```

---

## Installation

1. Drop the `Solived-AircraftFlares` folder into your `resources` directory.
2. Add `ensure Solived-AircraftFlares` to your `server.cfg`.
3. Open `config.lua` and add any custom aircraft to `Config.AllowedAircraft`.
4. If you want zone-locked rearming, verify or update the coords in `Config.Rearm.zones` for your map.
5. Start your server. The keybind registers automatically; players can rebind in GTA keybind settings.

No SQL, no exports to configure, no other resources required.

---

## Adding a Custom Aircraft

```lua
-- In Config.AllowedAircraft:
myfighter = { profile = 'fighter', ammo = 48, cooldownMs = 300, burstSize = 2, variant = 'gold' }
```

If the aircraft has a unique fuselage shape and the `fighter` offsets don't line up with the actual dispenser pods:

```lua
-- In Config.Profiles:
myfighter_pod = {
    offsets = {
        left  = { vec3(-2.1, -5.8, -0.30), vec3(-2.8, -6.1, -0.28) },
        right = { vec3( 2.1, -5.8, -0.30), vec3( 2.8, -6.1, -0.28) }
    }
}

-- Add defaults:
Config.DefaultProfileAmmo['myfighter_pod']     = 48
Config.DefaultProfileCooldown['myfighter_pod'] = 300
Config.DefaultProfileBurst['myfighter_pod']    = 2

-- Reference it:
myfighter = { profile = 'myfighter_pod' }
```

---

## Security Notes

- No client event can grant, restore, or modify ammo. All ammo state lives in `VehicleState` on the server.
- Every deploy and rearm event validates that the source player is currently inside the target vehicle and in an allowed seat. A spoofed `vehicleNetId` pointing to a vehicle the player isn't in is silently rejected.
- The client's `canDeploy()` is an input filter only. Its result is never transmitted to or trusted by the server.
- The server re-derives aircraft config from `GetEntityModel` on every request. A client sending a fake profile name or ammo count has no effect.
- The `/refillflares` command runs entirely server-side and optionally enforces ACE permissions. There is no client-side path to trigger an instant refill.
