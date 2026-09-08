# Testing & verification

## Continuous (every push / weekly) — GitHub Actions

`ci.yml` runs `esphome config` then a full `esphome compile` of all four
configs against the current ESPHome release. `secret-scan.yml` runs gitleaks
over the whole history. Badges are on the [README](README.md).

## Hardware bring-up — 2026-09-08, ESPHome 2026.8.1

**Rig:** the full pair only — `dehumidifier-controller.yaml` on a Wemos/LOLIN
ESP32 + soldered SHT41 (10 kΩ I²C pull-ups on the breakout), and
`dehumidifier-plug.yaml` on a **bare** Sonoff S31 Lite with **no load**. Both on
a phone hotspot. Driven and observed live through the `web_server` REST API
(`/events` stream, `POST /number/<name>/set`, `POST /switch/relay/...`).

| # | Check | Method | Result |
|---|---|---|---|
| 1 | Boot with no Home Assistant | power on, watch uptime | ✅ no reboot loop (`api: reboot_timeout: 0s`) |
| 2 | SHT41 read + dead-sensor safety | unplug / re-solder sensor | ✅ `Sensor fault` ON → desired forced OFF; clears when healthy |
| 3 | Wi-Fi stability | observe logs over minutes | ✅ steady after `power_save_mode: none` + `fast_connect` |
| 4 | Controller → plug reachability | set `Plug IP address`, watch `Plug reachable` | ✅ `OK` / `ON` once a real IP is set; `.local` correctly fails fast |
| 5 | Auth | REST calls with / without credentials | ✅ 401 unauthenticated; controller's generated `Basic` header accepted |
| 6 | Deadband OFF→ON | drop `RH on`/`RH off` below ambient RH | ✅ `Desired: ON`, plug relay closes |
| 7 | Deadband ON→OFF | raise the band back above ambient RH | ✅ `Desired: OFF`, relay opens |
| 8 | **Cool-off timer counts down** under the controller's 20 s idempotent `turn_off` re-asserts | watch `Cool-off remaining` after an ON→OFF | ✅ `45→…→6` monotonic, no reset *(this was the bug fixed in 127e3a3)* |
| 9 | Cool-off **blocks** a turn-on mid-window | force OFF then immediately want-ON | ✅ relay stays OFF, `HOLD-OFF (n s left)`, controller still gets `OK` (deferred) |
| 10 | Cool-off **releases** | wait out the window | ✅ relay closes once plug cool-off *and* controller `min_off` both clear |
| 11 | FORCE relay ON button | `POST .../press` | ✅ energizes immediately, bypasses cool-off / power-on delay |
| 12 | **Stale-command watchdog** | set timeout 1 min, force relay ON, **unpower the controller** | ✅ relay forced OFF ~65 s after the last command (60 s timeout + 10 s check) |
| 13 | Recovery after controller power cut | re-power | ✅ re-syncs within one cycle; `Plug IP address` persisted through the outage |
| 14 | Power-on delay | observed after each flash reboot | ✅ `Cool-off active` for the configured minutes, then clears (same code path as #8) |

### Not covered

- The two `*-simple.yaml` configs — compile + CI only, not bench-tested.
- Behaviour driving a **real compressor load** — nameplate / inrush / relay
  rating is the installer's responsibility, see
  [README → Load and relay rating](README.md#load-and-relay-rating).
- The watchdog at its shipped 15-minute timeout (tested at 1 minute — same
  `interval:` code, just the threshold changed).

## Re-running the key checks

With both devices on the LAN and `USER`/`PASS` = your `web_server` credentials:

```bash
BASE=http://<controller-ip>
# watch everything live
curl -u $USER:$PASS -N $BASE/events

# force a turn-on: set the band below current RH (set RH_off first when lowering)
curl -u $USER:$PASS -X POST "$BASE/number/RH%20off%20%20(turn%20OFF%20at%20or%20below%20%25RH)/set" -d value=30
curl -u $USER:$PASS -X POST "$BASE/number/RH%20on%20%20(turn%20ON%20at%20or%20above%20%25RH)/set"  -d value=35
```

The plug's cool-off / watchdog timers can be shortened the same way
(`/number/Plug%20min%20off.../set`, etc.) for faster iteration — **restore the
defaults afterwards** (controller 60/50/5/1, plug 5/5/15).
