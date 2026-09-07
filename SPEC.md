# ESPHome Two-Box Dehumidifier Controller — Project Spec

## Task for the coding agent
Produce two complete, compiling ESPHome YAML configurations plus a short README.
This is a standalone humidistat with **no Home Assistant and no MQTT**. The two
devices talk directly to each other over HTTP. Both expose the ESPHome web
server UI and a captive-portal onboarding flow (the target Wi-Fi network is
unknown at build time — the device will be handed to a third party).

Before writing YAML, verify anything marked **[VERIFY]** against current ESPHome
docs and a known-good Sonoff S31 configuration — several of these components’
options have changed across releases.

---

## Background / intent
A basement dehumidifier’s built-in control has almost no hysteresis: it restarts
within ~30 s of finishing a cycle, running near-continuously and short-cycling
the compressor. This controller replaces that with a **wide, adjustable
deadband** plus an **anti-short-cycle minimum-off timer**. Result: far fewer
compressor starts and lower energy use. A 5–15 %RH swing in the space is fully
acceptable; precision is explicitly *not* a goal.

## Hardware (on hand)
- **Box A — “Controller” (the brain):** Wemos/LOLIN **ESP32** (4 MB flash) +
  **Sensirion SHT41** on I²C. Low-voltage only; never touches mains. Powered
  from a listed USB supply.
- **Box B — “Plug” (the switch):** **Sonoff S31 Lite (Wi-Fi, ESP8285, ~1 MB
  flash)** — ETL-listed, carries the 120 VAC to the dehumidifier. Flashed via
  its internal serial header.
  - **[VERIFY] Confirm it is the Wi-Fi S31 Lite, not the Zigbee S31 Lite.**
    The Zigbee variant has no ESP chip, cannot be flashed, and makes this design
    impossible.
- **Load:** residential basement dehumidifier (compressor motor load). Assume
  running current ≤ ~12 A continuous (nameplate check happens outside this task).

## Roles & communication
- **ESP32 = brain.** Reads SHT41, runs hysteresis + timers, decides the desired
  relay state, and commands the plug.
- **S31 Lite = dumb actuator.** Exposes its relay; obeys commands; defaults to
  the safe (off) state; runs a stale-command watchdog.
- **Transport:** ESP32 → S31 via **HTTP `http_request`** hitting the S31
  web_server REST endpoints. No HA, no MQTT, no ESP-NOW (ESP-NOW is ESP32-only
  and the Lite is an ESP8266-class part).

## Deliverables
1. `dehumidifier-controller.yaml` (ESP32 + SHT41)
2. `dehumidifier-plug.yaml` (S31 Lite)
3. `README.md` — flashing order, captive-portal onboarding (two separate APs,
   one per device), how to set/adjust setpoints from the web UI, and how to
   point the controller at the plug (mDNS default + IP fallback).

---

## Requirements common to both devices
- Target current stable ESPHome.
- `wifi:` with an `ap:` fallback block **and** `captive_portal:` so each device
  raises its own AP + portal on first boot / when it can’t join a network. Use a
  known fallback AP password (document it in the README).
- `web_server:` **`version: 2`** — v2 embeds its assets in flash and works with
  no internet access. **Do not use v3** here: v3 loads its front-end JS from a
  CDN at page-load, which fails in an offline basement. (v2’s larger binary is
  the main flash-pressure point on the S31 — if it does not fit, that is the
  thing to trim first.)
- `api:` present **with `reboot_timeout: 0s`** (or omit `api:` entirely). Without
  this, the native API’s default ~15-minute no-client timeout will **reboot-loop
  both devices** because there is no Home Assistant connecting. This is the
  single most common mistake when running ESPHome standalone.
- **`web_server:` `auth:` (username + password) on BOTH devices.** This gates
  the web UI and the REST relay endpoint so a random device on the LAN can’t hit
  `/switch/relay/turn_on`. Understand its limits: web_server has no TLS, so this
  is HTTP Basic auth (base64 over plain HTTP) — access control, **not**
  on-the-wire encryption. Acceptable for a home LAN; do not treat it as
  confidentiality. Use the same credentials on both devices so the controller
  can authenticate to the plug. Store them via `!secret` (a `secrets.yaml`), not
  inline. Do not attempt HTTPS/TLS — the S31’s ESP8285 lacks the headroom and
  there is no LAN-local threat that justifies it.
- `ota:` enabled with a password. `logger:` on.
- Device `name:` sets the mDNS hostname — use `dehumidifier-controller` and
  `dehumidifier-plug`.

---

## `dehumidifier-plug.yaml` (Sonoff S31 Lite / ESP8285)
- Platform/board for ESP8285. **[VERIFY] GPIO map** (standard S31 / S31 Lite):
  - Relay: **GPIO12**
  - Button: **GPIO0**
  - Status LED (green/link): **GPIO13**, `inverted: true`
  - (No CSE7766 — the “Lite” omits power monitoring. No UART sensor.)
- **Relay control is gated through a template switch, not the bare GPIO** — this
  is what makes the cool-off a real backstop rather than advice:
  - Physical relay: `platform: gpio`, `pin: GPIO12`, `id: relay_gpio`,
    `internal: true`, **`restore_mode: ALWAYS_OFF`** (safe state on power-up).
  - Exposed control: a **template `switch`** `id: relay`, `name: "Relay"` — what
    `web_server` and the button command. Its `turn_on_action` runs the cool-off
    gate (below) before energizing `relay_gpio`; `turn_off_action` opens
    `relay_gpio` immediately and stamps the last-off time.
- Physical button (GPIO0) → toggles the template `relay` (so the button obeys
  cool-off too).
- Status LED (GPIO13, `inverted: true`) tracks the *physical* relay
  (`relay_gpio`).
- `web_server` (with `auth:`) exposes `relay` for authenticated REST control:
  - `POST http://<plug>/switch/relay/turn_on`
  - `POST http://<plug>/switch/relay/turn_off`  (turn-off is never gated)
  - Both require HTTP Basic auth (see common requirements).

- **Minimum-off / compressor cool-off (REQUIRED, non-overridable on the normal
  paths):**
  - After the relay turns **off**, refuse any turn-**on** for `PLUG_MIN_OFF`
    (plug-side `number`, `restore_value`, shown on the plug page; default 5 min,
    range 1–30). Turn-**off** is always immediate and never gated.
  - Applies to **every** normal turn-on path — controller REST commands, the
    physical button, and the boot re-assert. The controller path and the button
    **cannot** override it. Rationale: restarting a compressor before refrigerant
    pressures equalize risks a high-head start, inrush, overload trip, and wear —
    physics, not policy.
  - A turn-on during the window is **deferred, not queued**: rejected and flagged
    (see hold-off indication). The controller’s next idempotent re-assert (~20 s)
    succeeds once the window expires — no explicit queue needed.

- **Power-cycle behavior (READ CAREFULLY — real limitation):** the dehumidifier
  is powered *through* the S31, so any S31 power loss also cuts the compressor.
  At every boot the compressor has just lost power, but the plug **cannot know
  how long it will have been off** before the next on-command. An ESP without a
  synced real-time clock cannot measure wall-clock elapsed time across a reboot
  (`millis()`/uptime resets to 0), so **a persisted `millis()` timestamp is
  meaningless after a power cycle — do not rely on it.** Therefore:
  - On boot, conservatively apply `POWER_ON_DELAY` (plug-side `number`,
    `restore_value`, default = `PLUG_MIN_OFF`, range 0–30 min) before allowing
    the first turn-on. Protects the brief-blip case (compressor running seconds
    ago), which is the most dangerous restart.
  - **Operator escape hatch for the long-rest case:** if the compressor has
    actually been off a long time (unit powered down overnight) and the operator
    doesn’t want to wait out the boot delay, the labeled **force/bypass** control
    (below) starts it immediately. This is the intended, deliberate override for
    the "power-cycled after a long rest, don’t make me wait" case.
  - OPTIONAL auto-resolution: if `time:` (SNTP) is configured *and* actually
    syncs (needs internet — not guaranteed in a basement), the plug may persist
    the wall-clock last-off time and, on boot, skip/shorten the delay when real
    elapsed off-time already exceeds `PLUG_MIN_OFF`. Enhancement only; the design
    must be correct with no clock.

- **Force-relay / bypass control (OPTIONAL, plug page ONLY):** a clearly-labeled
  template `button`/`switch` — e.g. "Force relay ON (bypass cool-off)" — that
  energizes `relay_gpio` immediately regardless of cool-off / power-on delay.
  Present **only** on the plug’s own web page, never on the controller path, and
  labeled with a compressor-wear warning. For commissioning/testing and the
  long-rest power-cycle case above.

- **Hold-off indication on the plug page (REQUIRED):** expose the cool-off state
  in `web_server` so the operator can see *why* the relay is off —
  a `binary_sensor` "Cool-off active", a numeric `sensor` "Cool-off remaining
  (s)" counting down, and/or a `text_sensor` "Relay status" reading e.g. `ON`,
  `OFF`, `HOLD-OFF (192 s left)`.

- **Stale-command watchdog (REQUIRED safety):** if no command has been received
  from the controller within **`STALE_TIMEOUT` (default 15 min)**, force the
  relay **off**. Rationale: if the controller dies, a dehumidifier stuck *on*
  runs the compressor unattended — the mains side must fail safe on its own.
  - Suggested implementation: a `global` last-command timestamp refreshed on
    every inbound control action, and an `interval:` that opens the relay when
    the age exceeds `STALE_TIMEOUT`. **[VERIFY]** that same-state re-asserts still
    refresh the watchdog.

## `dehumidifier-controller.yaml` (ESP32 + SHT41)
- `i2c:` on the board’s default pins (**[VERIFY]** Wemos ESP32 D1 mini: SDA
  GPIO21 / SCL GPIO22).
- `sensor:` `platform: sht4x`, address `0x44`, sampled every `SAMPLE_INTERVAL`
  (default 20 s). Expose humidity (%RH) and temperature.
- **User-adjustable config, all shown in web_server and all
  `restore_value: true` (survive reboot), no recompile needed to change:**
  - `number` **RH_on** — turn-on setpoint %RH (default 60, range 30–80)
  - `number` **RH_off** — turn-off setpoint %RH (default 50, range 25–75)
  - `number` **min_off_minutes** — compressor anti-short-cycle (default 5,
    range 1–30)
  - `number` **min_on_minutes** — optional minimum on-time (default 1, range 0–30)
  - `text` **s31_target** — hostname/IP the commands are sent to
    (default `dehumidifier-plug.local`)
  - Enforce **RH_off < RH_on** (clamp or reject otherwise) so a real deadband
    always exists.
- **Control loop (evaluate every `SAMPLE_INTERVAL`):**
  1. Hysteresis: `RH >= RH_on` ⇒ want **ON**; `RH <= RH_off` ⇒ want **OFF**;
     in-between ⇒ **hold** current desired state.
  2. Gate OFF→ON transition behind `min_off_minutes` since the last OFF.
     Optionally gate ON→OFF behind `min_on_minutes`.
  3. **Re-assert the desired state to the plug every cycle (idempotent),** not
     only on change — so a plug reboot re-syncs within one cycle. Build the URL
     from `s31_target` and POST to the matching `/switch/relay/turn_on|off`.
     Include the HTTP Basic auth credentials (same secret as the plug’s
     `web_server: auth:`) — as an `Authorization: Basic <base64>` header or via
     `http_request`’s auth support. **[VERIFY]** current `http_request` action
     syntax, that its `url:` accepts a lambda/template (needed to build the URL
     from `s31_target`), and the current mechanism for supplying Basic auth.
  4. **Capture and classify each command’s outcome (REQUIRED).** Use
     `http_request` response capture to read the HTTP **status code** and any
     body, and map the result to a human-readable last-command status:
     - `OK` — 2xx.
     - `AUTH FAILED` — 401/403 (wrong/missing Basic auth credentials).
     - `UNREACHABLE` — connection refused / timeout / DNS-or-mDNS resolve
       failure (bad `s31_target`, plug down, network drop).
     - `HTTP ERROR <code>` — any other non-2xx.
     On anything other than `OK`: log the cause, keep local desired state, and
     retry next cycle (the periodic re-assert is the retry).
     **[VERIFY]** current `http_request` response-capture syntax (status code +
     body) for the target ESPHome version.
     - NOTE on the plug’s cool-off: a turn-on the plug *defers* during its
       hold-off window still returns HTTP 2xx, so transport status alone reads
       `OK` even though the relay didn’t close. Distinguishing "deferred by
       cool-off" from "actually on" requires reading the plug’s relay/hold-off
       state — that is the OPTIONAL read-back below, left optional per decision.
- **Safety:**
  - Boot desired state = **OFF**.
  - If SHT41 reads invalid/NaN for **N consecutive cycles (default 3)** ⇒ command
    **OFF** (never run blind on a dead sensor).
- **Web UI main page** should surface: current RH & temp, desired state, the
  **last-command outcome** (`OK` / `AUTH FAILED` / `UNREACHABLE` /
  `HTTP ERROR <code>`) with its timestamp, time-in-current-state, and the
  editable setpoints. Optional `binary_sensor` “plug reachable” derived from the
  last HTTP result. Optional read-back: GET the plug’s relay/hold-off state to
  confirm actual vs. commanded and report "deferred – plug in cool-off".

---

## Addressing / mDNS caveat — call out in the README
`http_request` to a `.local` mDNS name is **not reliably resolvable** on ESP.
Design accordingly: `s31_target` defaults to `dehumidifier-plug.local` (works
when mDNS resolution does), and can be overridden with the plug’s IP address if
it doesn’t. README should instruct: read the plug’s assigned IP from the plug’s
own web page after onboarding, and enter it into the controller’s `s31_target`
field. No reflash required.

## Onboarding flow (README)
1. Flash both devices over serial (plug requires opening the case + serial
   header; controller over USB).
2. Power each; each raises its own fallback AP. Join **each** AP in turn and
   enter the target Wi-Fi credentials via captive portal (two APs to onboard —
   expected).
3. Open each device’s web page. Note the plug’s IP.
4. On the controller page, confirm/adjust setpoints; set `s31_target` to the
   plug’s `.local` name (default) or its IP if needed.

## Constraints / non-goals
- No Home Assistant, no MQTT, no cloud, no external broker. Two devices only.
- Must run fully standalone on an **unknown** Wi-Fi network with possibly **no
  internet**.
- All 120 VAC switching is done by the **listed** S31 Lite; the ESP32/SHT41 side
  is low-voltage only.
- Power/energy monitoring is out of scope (the Lite has no metering hardware).

## Acceptance criteria
- Both YAMLs compile for their respective targets.
- With no Home Assistant present, **neither device reboot-loops**.
- Cold boot: plug relay is off; controller asserts the correct state within one
  `SAMPLE_INTERVAL`.
- Crossing `RH_on` turns the dehumidifier on (subject to `min_off_minutes`);
  crossing `RH_off` turns it off.
- Killing the controller results in the plug turning **off** within
  `STALE_TIMEOUT`.
- Setpoints are editable from the web UI and persist across a reboot.
- Both web UIs require the configured username/password, and the controller
  authenticates successfully to the plug’s REST endpoint (an unauthenticated
  request to `/switch/relay/turn_on` is rejected).
- A plug reboot (simulated power blip) re-syncs to the controller’s desired
  state within one cycle without manual intervention.
- After the relay turns off, a turn-on within `PLUG_MIN_OFF` is refused by the
  plug and shown as hold-off on the plug page; it succeeds once the window
  expires. The controller path and the button cannot bypass it.
- After a plug power-cycle, the relay stays off for `POWER_ON_DELAY` before any
  first turn-on is honored; the plug-page force/bypass control (if built) starts
  it immediately.
- The controller’s web page shows the correct last-command outcome for each of:
  normal success, wrong credentials (`AUTH FAILED`), and a bad/unreachable
  `s31_target` (`UNREACHABLE`).
