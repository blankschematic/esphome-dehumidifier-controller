# Two-Box Dehumidifier Controller (ESPHome, no Home Assistant)

[![validate](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/validate.yml/badge.svg)](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/validate.yml)

A standalone humidistat for a basement dehumidifier whose built-in control has
almost no hysteresis and short-cycles the compressor. This replaces it with a
**wide adjustable deadband** plus an **anti-short-cycle minimum-off timer**.

Two ESP devices talk directly to each other over HTTP — **no Home Assistant, no
MQTT, no cloud**. Each raises its own Wi-Fi AP + captive portal on first boot,
so it can be onboarded onto an unknown network.

```
  ┌─────────────────────────┐        HTTP POST         ┌──────────────────────────┐
  │  Controller  (Box A)    │  /switch/relay/turn_on   │   Plug  (Box B)          │
  │  Wemos ESP32 + SHT41    │ ───────────────────────► │   Sonoff S31 Lite        │
  │  reads RH, runs the     │  /switch/relay/turn_off  │   switches 120 VAC to    │
  │  deadband + timers      │ ◄─────────────────────── │   the dehumidifier      │
  │  low voltage only       │      200 / 401 / ...     │   ETL-listed, fail-safe  │
  └─────────────────────────┘                          └──────────────────────────┘
```

---

## Two tiers — pick one per box, matched

| File | Use |
|---|---|
| `dehumidifier-controller-simple.yaml` / `dehumidifier-plug-simple.yaml` | **Teaching / first explanation.** One self-contained file each. Deadband + min-off timer + idempotent re-assert, and nothing else. No auth, no plug-side safety. |
| `dehumidifier-controller.yaml` / `dehumidifier-plug.yaml` | **Full.** Everything below. Split into small reusable `packages:` (Wi-Fi, web server, board pinout) that both boxes share. |

Run a **matched pair** — the simple controller has no auth and talks only to
the simple (no-auth) plug; the full controller sends HTTP Basic auth and talks
only to the full plug.

### What the guard rails add (simple → full)

| Concern | Simple | Full |
|---|---|---|
| Deadband hysteresis (`RH_on` / `RH_off`) | ✅ | ✅ |
| Anti-short-cycle min-off (controller side) | ✅ | ✅ |
| Idempotent re-assert every cycle | ✅ | ✅ |
| HTTP Basic auth on the relay endpoint | ❌ | ✅ |
| Command outcome classification (`OK` / `AUTH FAILED` / `UNREACHABLE` / `HTTP ERROR n`) | ❌ (`OK` / `unreachable` only) | ✅ |
| Dead-sensor lockout (N bad reads ⇒ force OFF) | ❌ (holds state) | ✅ |
| Minimum on-time | ❌ | ✅ |
| **Plug-side** compressor cool-off (non-overridable on normal paths) | ❌ | ✅ |
| **Plug-side** power-on delay after a power cut | ❌ | ✅ |
| **Plug-side** stale-command watchdog (controller dies ⇒ relay off) | ❌ | ✅ |
| Hold-off status sensors + force/bypass button on the plug page | ❌ | ✅ |

---

## Hardware

- **Box A — Controller:** Wemos/LOLIN **ESP32** (4 MB) + Sensirion **SHT41** on
  I²C (SDA `GPIO21`, SCL `GPIO22`, address `0x44`). USB-powered. Never touches
  mains.
- **Box B — Plug:** **Sonoff S31 Lite (Wi-Fi, ESP8285, ~1 MB).**
  Relay `GPIO12`, button `GPIO0`, green LED `GPIO13` (inverted).
  - ⚠️ Must be the **Wi-Fi** S31 Lite. The **Zigbee** S31 Lite has no ESP chip,
    cannot be flashed, and makes this design impossible.
  - The "Lite" has no CSE7766 — no power/energy monitoring (out of scope).

---

## What you need

- **Box A:** a Wemos/LOLIN **ESP32** dev board + a Sensirion **SHT4x** breakout
  (SHT41 or SHT40), four jumper wires, a case, a USB power supply.
- **Box B:** a **Sonoff S31 Lite** (Wi-Fi — see the hardware warning above).
- A **3.3 V USB-to-serial adapter** (CP2102 / CH340 / FT232) to flash the S31
  the first time — plus a way to reach its 4 pads (pogo-pin jig or soldered
  header). The case must be opened.
- A **USB cable** for the ESP32.
- A computer with **Python 3.11 or newer** (3.12 is the safest — some 3.13
  setups still hit a dependency-wheel issue on Windows).

## 1. Get the files

```bash
git clone https://github.com/blankschematic/esphome-dehumidifier-controller.git
cd esphome-dehumidifier-controller
```

Keep the `packages/` folder next to the configs — the full configs `!include`
from it. (If you only want the teaching version, the two `*-simple.yaml` files
stand alone and need nothing else.)

## 2. Install ESPHome

```bash
pipx install esphome            # recommended; or:  pip install --user esphome
```

Alternatives: the [ESPHome Docker image](https://esphome.io/guides/getting_started_command_line),
or the ESPHome add-on if you happen to run Home Assistant (this project does not
require it). Check it works:

```bash
esphome version                 # expect 2026.8.0 or newer
```

> **Windows:** run `esphome` from **PowerShell** or **Command Prompt**, not
> Git Bash / MSYS. The ESP32 build uses ESP-IDF, whose toolchain fails silently
> under an MSYS shell and produces no firmware.

## 3. Create `secrets.yaml`

```bash
cp secrets.yaml.example secrets.yaml     # Windows: copy secrets.yaml.example secrets.yaml
```

Edit it:

| Key | Used by | Notes |
|---|---|---|
| `wifi_ssid`, `wifi_password` | all | the Wi-Fi the devices join after onboarding |
| `fallback_ap_password` | all | password for the setup AP each device raises on first boot (≥ 8 characters) |
| `ota_password` | all | protects wireless firmware updates |
| `web_server_username`, `web_server_password` | full only | one login, used on **both** boxes. Gates the web page **and** the relay REST endpoint. The controller builds its `Authorization: Basic …` header from these automatically — nothing to encode by hand. Keep the password free of `"` `\` `$`. |

## 4. Compile and flash

Flash the **plug first**, over serial, **with the S31 unplugged from mains**:

1. Wire the USB-serial adapter to the S31's header — `3V3 / GND / RX↔TX / TX↔RX`.
2. Hold the S31 button while plugging in the USB-serial adapter — this enters
   the ESP8285's flash mode.
3. Run, and pick the adapter's serial port when prompted:
   ```bash
   esphome run dehumidifier-plug.yaml
   ```

Then the **controller**. First wire the SHT4x to the ESP32 —
`VIN/VCC → 3V3`, `GND → GND`, `SDA → GPIO21`, `SCL → GPIO22` — then flash over
the USB cable:

```bash
esphome run dehumidifier-controller.yaml
```

(The board logs an I²C error and holds the relay OFF until the sensor is wired
and reads a real humidity — that's the dead-sensor safety working.)

After this first serial flash, both devices accept **wireless (OTA) updates** —
`esphome run …` will offer the network target automatically once they're on
your Wi-Fi.

> To browse/edit configs in a local web UI instead of the CLI, run
> `esphome dashboard .` in this folder and open <http://localhost:6052>.

**Verified with ESPHome 2026.8.1** — all four configs compile. Flash use: plug
43 %, controller 56 %. Both leave ample room; `web_server v2 local` fits the
S31 Lite fine.

> The two controller configs share the mDNS name `dehumidifier-controller`
> (likewise the two plug configs), so they also share an
> `.esphome/build/…` directory — switching between simple and full triggers a
> clean rebuild. Harmless; you only ever flash one of each pair.

### If the plug ever overflows flash (not currently an issue)

At 43 % it isn't close. If a future ESPHome release changes that,
`web_server: version: 2` + `local: true` is the biggest consumer and the first
thing to trim:

1. Drop `local: true` from `packages/web-server.yaml` (keeps `version: 2`; the plug's
   *page* then needs internet to load its JS, but the **REST endpoint still
   works fully offline** — that's all the controller uses).
2. Or drop the `web_server` package from the plug entirely and rely on the
   controller's telemetry. You lose the on-device hold-off display.
3. `esphome compile` reports the flash figure; OTA needs the image under
   roughly half of flash.

---

## Onboarding (each device, once)

Both devices ship not knowing your Wi-Fi.

1. Power the device. It fails to join and raises its own AP:
   - Controller → SSID **`dehumidifier-controller`**
   - Plug → SSID **`dehumidifier-plug`**
   - password = your `fallback_ap_password`
2. Join that AP with a phone/laptop. A captive portal opens (or browse
   `http://192.168.4.1`). Enter your real Wi-Fi SSID + password. The device
   reboots and joins.
3. Repeat for the other device. **Two APs to onboard — expected.**
4. Open each device's web page (see addressing below) and enter the
   `web_server` username/password (full configs).

---

## Addressing: mDNS vs IP  — important

`http_request` to a `.local` mDNS name is **not reliably resolvable on ESP**.

- The controller's **`Plug target`** field defaults to
  `dehumidifier-plug.local`. That works *when* mDNS resolution happens to work.
- If the controller's `Last command outcome` shows **`UNREACHABLE`** even
  though the plug is up:
  1. Open the **plug's** own web page, read its **`IP Address`** sensor
     (e.g. `192.168.1.57`).
  2. On the **controller's** page, set **`Plug target`** to that IP.
  3. No reflash. Consider a DHCP reservation for the plug so the IP is stable.

Find each device's page from another machine on the LAN:
`http://dehumidifier-controller.local` / `http://dehumidifier-plug.local`, or
by IP from your router's client list.

---

## Using it

On the **controller** page:

| Field | Default | Range | Meaning |
|---|---|---|---|
| `RH on` | 60 % | 30–80 | at/above this, want the dehumidifier ON |
| `RH off` | 50 % | 25–75 | at/below this, want it OFF (auto-clamped below `RH on`) |
| `Min off minutes` | 5 | 1–30 | compressor rest before another start |
| `Min on minutes` | 1 | 0–30 | *(full only)* minimum run once started |
| `Plug target` | `dehumidifier-plug.local` | — | hostname or IP of the plug |

Between `RH off` and `RH on` the controller **holds** whatever it last decided —
that band is the whole point; a 5–15 %RH swing in the room is fine.

On the **plug** page (full config):

| Field | Default | Meaning |
|---|---|---|
| `Plug min off` | 5 min | turn-**on** refused for this long after any turn-off. **Not overridable** by the controller or the button. |
| `Power-on delay` | 5 min | after the plug itself loses power, the first turn-on is refused for this long (an ESP without a synced clock can't measure real off-time across a reboot, so it assumes worst case) |
| `Stale-command timeout` | 15 min | no command from the controller for this long ⇒ relay forced OFF |
| `Relay status` | — | `ON` / `OFF` / `HOLD-OFF (192 s left)` |
| `Cool-off remaining` | — | seconds left on the longest active hold-off |
| **`FORCE relay ON (bypass cool-off)`** | — | operator escape hatch for "powered down overnight, don't make me wait". ⚠️ compressor wear if used to bypass a real cool-off. |

A turn-on the plug **defers** during a hold-off still returns **HTTP 200** — the
controller's periodic re-assert (~20 s) simply succeeds once the window
expires. No queue.

---

## How it behaves (acceptance checks)

- **Cold boot:** plug relay off; controller asserts the correct state within one
  `sample_interval` (20 s).
- **No Home Assistant:** neither device reboot-loops (`api: reboot_timeout: 0s`).
- Crossing `RH on` turns the dehumidifier on (after `Min off minutes`); crossing
  `RH off` turns it off.
- **Kill the controller:** plug forces itself off within `Stale-command timeout`
  *(full)*.
- **Plug power blip:** on reboot the plug comes up off and re-syncs to the
  controller's desired state within one cycle. *(Full)* the first turn-on waits
  out `Power-on delay` unless you press force.
- Setpoints persist across reboot.
- *(Full)* an unauthenticated `POST /switch/relay/turn_on` to the plug is
  rejected; the controller's page shows `AUTH FAILED` for wrong credentials and
  `UNREACHABLE` for a bad `Plug target`.

---

## Security note

`web_server` has no TLS. The `auth:` on the full configs is **HTTP Basic auth
(base64 over plain HTTP)** — access control so a random LAN device can't flip
the relay, **not** on-the-wire encryption. Fine for a home LAN; the S31's
ESP8285 has no headroom for HTTPS and there's no LAN-local threat that needs it.
The **simple** configs have no auth at all — they're for bench explanation, not
for leaving on a shared network.

## Disclaimer

The S31 Lite is an ETL-listed appliance that switches mains voltage. Flash it
**disconnected from mains**, don't modify its high-voltage side, and reassemble
the case before use. Match the dehumidifier's running current against the S31's
rating. This project is shared as-is, with no warranty; you are responsible for
safe installation and for the compressor-protection timings you set.

---

## Repository layout

```
dehumidifier-controller-simple.yaml   standalone, teaching
dehumidifier-plug-simple.yaml         standalone, teaching
dehumidifier-controller.yaml          full  — pulls in packages/
dehumidifier-plug.yaml                full  — pulls in packages/

packages/
  network.yaml               wifi + ap + captive_portal + api reboot_timeout:0s + ota + logger
  web-server.yaml            web_server v2, local, basic auth
  board-esp32-sht41.yaml     ESP32 board + i2c + sht4x + http_request
  board-sonoff-s31-lite.yaml S31 Lite board + internal relay_gpio + button + LED

secrets.yaml.example         copy to secrets.yaml
SPEC.md                      original design brief / rationale
```

The files in `packages/` are ESPHome **packages**, `!include`d by the
`packages:` block at the top of each full config. The `-simple` files
intentionally inline the same content so each is a single copy-pasteable file
with nothing to include.
