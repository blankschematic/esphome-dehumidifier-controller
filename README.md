# Two-Box Dehumidifier Controller (ESPHome, no Home Assistant)

[![ci](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/ci.yml/badge.svg)](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/ci.yml)
[![secret-scan](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/secret-scan.yml/badge.svg)](https://github.com/blankschematic/esphome-dehumidifier-controller/actions/workflows/secret-scan.yml)

A basement dehumidifier's built-in humidistat has almost no hysteresis — it
restarts within ~30 s of finishing a cycle and short-cycles the compressor.
This replaces it with a **wide, adjustable deadband** plus an
**anti-short-cycle minimum-off timer**: far fewer compressor starts, lower
energy use. A 5–15 %RH swing in the room is fine — precision is not the goal.

Two small ESP boxes do it between themselves over your Wi-Fi — **no Home
Assistant, no MQTT, no cloud**. Each raises its own Wi-Fi hotspot on first boot
for onboarding, so it works on a network it has never seen.

```text
 SHT41  ──I²C──▶  ESP32  ──Wi-Fi──▶  Sonoff S31  ──mains──▶  Dehumidifier
 humidity         Box A               Box B
                  reads RH,           switches the load,
                  runs the logic      enforces compressor limits
```

## Quick start

- **Parts:** a Wemos/LOLIN ESP32 + an SHT4x breakout, a **Wi-Fi** Sonoff S31 or
  S31 Lite, a 3.3 V USB-serial adapter, and a computer with Python.
  ([full list](#what-you-need)) — **check the relay can take your dehumidifier's
  load** ([why](#load-and-relay-rating)).
- **Build and flash:**
  ```bash
  pip install esphome
  git clone https://github.com/blankschematic/esphome-dehumidifier-controller.git
  cd esphome-dehumidifier-controller
  cp secrets.yaml.example secrets.yaml         # then edit it
  esphome run dehumidifier-plug.yaml           # USB-serial, S31 off mains
  esphome run dehumidifier-controller.yaml     # USB
  ```
  ([step by step](#1-get-the-files))
- **Two versions:** the `*-simple.yaml` files are for learning — one file each,
  no guard rails. The plain-named ones are for real use (compressor protection,
  auth, watchdog). Run a **matched pair**.
- **New to ESPHome?** Read `dehumidifier-plug-simple.yaml` first — it's ~40
  lines and does the whole job minus the safety nets.

---

## How it works

The **controller** samples the SHT41 every 20 s, applies the deadband and the
timers, and each cycle **re-sends** the desired relay state to the plug (not
only on a change) — so a plug reboot re-syncs within one cycle. It reads the
HTTP status back and shows `OK` / `AUTH FAILED` / `UNREACHABLE` / `HTTP ERROR`
on its own web page.

The **plug** exposes its relay over a REST endpoint and obeys — but it enforces
its own compressor cool-off, a power-on delay, and a watchdog that cuts the
relay if the controller goes silent. Turn-**off** is always immediate;
turn-**on** is what the safety timers gate.

```text
  ┌─────────────────────────┐     POST /switch/relay/turn_on      ┌─────────────────────────┐
  │  Controller  (Box A)    │     POST /switch/relay/turn_off     │   Plug  (Box B)         │
  │  Wemos ESP32 + SHT41    │ ─────────────────────────────────▶  │   Sonoff S31 / S31 Lite │
  │  deadband + timers,     │ ◀─────────────────────────────────  │   relay + cool-off,     │
  │  low voltage only       │     200 / 401 / timeout             │   watchdog, fail-safe   │
  └─────────────────────────┘                                     └─────────────────────────┘
```

---

## Two versions

Run a **matched pair**: the simple controller has no auth and talks only to the
simple (no-auth) plug; the full controller sends HTTP Basic auth and talks only
to the full plug.

<details>
<summary><b>What the full version adds over simple</b></summary>

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

</details>

---

## Hardware

- **Box A — Controller:** Wemos/LOLIN **ESP32** (4 MB) + Sensirion **SHT41** on
  I²C (SDA `GPIO21`, SCL `GPIO22`, address `0x44`). USB-powered. Never touches
  mains.
- **Box B — Plug:** a **Wi-Fi Sonoff S31 *or* S31 Lite** (ESP8285, ~1 MB). Same
  pinout on both — relay `GPIO12`, button `GPIO0`, green LED `GPIO13` (inverted).
  The full S31 has a CSE7766 power meter; this project simply leaves it
  unconfigured. The Lite omits it. Either works — power monitoring is not used.
  - ⚠️ It must be a **Wi-Fi** model. The **Zigbee** S31 / S31 Lite has no ESP
    chip, cannot be flashed, and makes this design impossible.
  - **Why a Sonoff and not a bare relay board:** the S31 is an **ETL-listed**
    (UL-equivalent) appliance. The mains switching, fusing, spacing, and
    enclosure are all inside a certified product — you never wire, solder, or
    expose 120 VAC yourself. The ESP32 + sensor side is low-voltage only.

---

## Load and relay rating

**Check this before you plug the dehumidifier in.** The S31's relay is rated
for a **resistive** load (heaters, lamps). A dehumidifier is a **compressor
(motor) load**, which is harder on relay contacts:

- **Inrush / locked-rotor current.** When the compressor starts it briefly
  draws several times its running current. A relay that easily carries the
  *running* current can still pit or weld its contacts on repeated inrush.
- **Motor derating.** Relay makers publish a lower rating for motor/inductive
  loads than the headline resistive figure.

So, before connecting the load:

1. Read the dehumidifier's **nameplate running current** (amps / FLA). This
   design assumes **≤ ~12 A continuous** on a 120 VAC circuit.
2. Confirm that against the **S31's own label** — and, if you can find it, the
   relay part's **motor-load** rating, not just its resistive rating.
3. If the unit is anywhere near the limit, or you can't establish its inrush,
   don't switch it directly. Drive a properly-rated contactor from the S31
   instead (outside this project's scope).

The anti-short-cycle timer helps here too: fewer make/break cycles under load
means slower contact wear over the relay's life.

---

## What you need

- **Box A:** a Wemos/LOLIN **ESP32** dev board + a Sensirion **SHT4x** breakout
  (SHT41 or SHT40), four jumper wires, a case, a USB power supply.
- **Box B:** a **Wi-Fi Sonoff S31 or S31 Lite** (see the hardware notes and
  [Load and relay rating](#load-and-relay-rating) above).
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

<details>
<summary><b>Build notes — flash headroom, shared build dir, verified version</b></summary>

**Verified with ESPHome 2026.8.1** — all four configs compile (CI), and the
full pair passed an end-to-end hardware test ([TESTING.md](TESTING.md)). Flash
use: plug 43 %, controller 54 %. Both leave ample room; `web_server v2 local`
fits the S31 Lite fine.

The two controller configs share the mDNS name `dehumidifier-controller`
(likewise the two plug configs), so they also share an `.esphome/build/…`
directory — switching between simple and full triggers a clean rebuild.
Harmless; you only ever flash one of each pair.

**If the plug ever overflows flash** (at 43 % it isn't close, but a future
ESPHome release could change that): `web_server: version: 2` + `local: true` is
the biggest consumer and the first thing to trim —

1. Drop `local: true` from `packages/web-server.yaml` (keeps `version: 2`; the
   plug's *page* then needs internet to load its JS, but the **REST endpoint
   still works fully offline** — that's all the controller uses).
2. Or drop the `web_server` package from the plug entirely and rely on the
   controller's telemetry. You lose the on-device hold-off display.
3. `esphome compile` reports the flash figure; OTA needs the image under
   roughly half of flash.

</details>

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

## Addressing: use the plug's IP — important

The controller **does not do mDNS lookups**. Resolving a `.local` name from an
ESP is slow to fail and was stalling the control loop hard enough to reboot the
device; the firmware disables resolver mDNS on purpose. The **`Plug IP
address`** field must be an IP — a `.local` name won't resolve.

1. Open the **plug's** own web page, read its **`IP Address`** sensor
   (e.g. `192.168.1.57`).
2. On the **controller's** page, set **`Plug IP address`** to that IP.
3. No reflash. Set a **DHCP reservation** for the plug so the IP is stable.

Until you do, `Last command outcome` reads `UNREACHABLE` and the relay stays
off — which is the safe state.

(This only affects the controller *resolving* the plug. Both devices still
**announce** themselves, so `http://dehumidifier-controller.local` /
`http://dehumidifier-plug.local` from your browser are unaffected.)

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
| `Plug IP address` | `192.168.1.2` | — | the plug's IP — **not** a `.local` name ([why](#addressing-use-the-plugs-ip--important)) |

Between `RH off` and `RH on` the controller **holds** whatever it last decided —
that band is the whole point; a 5–15 %RH swing in the room is fine.

`RH off` is always kept at least 1 below `RH on` (no inverted deadband). If you
move the band a long way, set the value you're raising **first**: widening up,
set `RH on` before `RH off`; widening down, set `RH off` before `RH on` —
otherwise the one you set first gets clamped against the old value of the other.

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

## What to expect

- **Cold boot:** the plug relay is off; the controller sets the right state
  within ~20 s.
- **No Home Assistant on the network:** neither box reboot-loops.
- RH climbs past `RH on` → the dehumidifier runs, once `Min off minutes` allows.
  RH drops past `RH off` → it stops.
- **Full:** if the controller loses power or crashes, the plug cuts the relay
  within `Stale-command timeout`.
- **Full:** after a power blip the plug comes back off and waits out
  `Power-on delay` before the first restart (or press **Force**).
- Setpoints survive a reboot.
- **Full:** the relay endpoint rejects unauthenticated requests. Wrong
  credentials show as `AUTH FAILED` and a bad `Plug IP address` as `UNREACHABLE` on
  the controller page.

Every item above was checked on real hardware — see [TESTING.md](TESTING.md)
for the method and results, and what wasn't covered.

---

## Troubleshooting

**Controller log: `sht4x: Communication failed` / I²C scan `Found no devices`.**
Nothing is acknowledging on the I²C bus. The ESP32's internal pull-ups are
enabled but only ~45 kΩ — too weak for I²C on anything but the shortest
wiring. In order of likelihood: the SHT4x breakout has no pull-ups and needs
external **4.7 kΩ from SDA→3V3 and SCL→3V3**; SDA/SCL are swapped; a cold
solder joint; the sensor isn't getting ~3.3 V (measure at its pins — never feed
it 5 V). Wired to pins other than GPIO21/22? Set
`substitutions: { i2c_sda_pin: GPIOxx, i2c_scl_pin: GPIOyy }`. While the sensor
is unhealthy the controller holds the dehumidifier **off** — that's the
dead-sensor safety, not a bug.

**Controller: `Last command outcome` = `UNREACHABLE`.** Set `Plug IP address` to the
plug's **IP**, not `.local` — see [Addressing](#addressing-use-the-plugs-ip--important).

**Wi-Fi log: `Authentication Failed`, then connects on the retry.** Harmless
noise from ESP32 power-save; the configs set `power_save_mode: none` +
`fast_connect: true` to suppress it. If it persists, your AP may be WPA3-only
with a fussy transition mode.

**Either device reboots every ~20–30 s (`task_wdt` / `Unsuccessful boot
attempts` climbing).** A blocking network call was tripping the task watchdog.
Fixed in firmware (resolver mDNS disabled, watchdog window widened, requests
gated on Wi-Fi) — reflash if you're on an older build.

---

## Security note

`web_server` has no TLS. The `auth:` on the full configs is **HTTP Basic auth
(base64 over plain HTTP)** — access control so a random LAN device can't flip
the relay, **not** on-the-wire encryption. Fine for a home LAN; the S31's
ESP8285 has no headroom for HTTPS and there's no LAN-local threat that needs it.
The **simple** configs have no auth at all — they're for bench explanation, not
for leaving on a shared network.

## Disclaimer

The S31 is an ETL-listed appliance that switches mains voltage. Flash it
**disconnected from mains**, don't modify its high-voltage side, and reassemble
the case before use. Verify the load against
[Load and relay rating](#load-and-relay-rating). This project is shared as-is,
with no warranty; you are responsible for safe installation and for the
compressor-protection timings you set.

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
TESTING.md                   CI + hardware verification, what was and wasn't checked
```

The files in `packages/` are ESPHome **packages**, `!include`d by the
`packages:` block at the top of each full config. The `-simple` files
intentionally inline the same content so each is a single copy-pasteable file
with nothing to include.
