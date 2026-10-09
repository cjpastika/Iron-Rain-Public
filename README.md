# Iron Rain

**MAVLink Exploitation Framework**

> Status: **Phase 4 — real-autopilot transport, hardware-validated over SiK.**
> The Phase 3 ESP32 man-in-the-middle — runtime attack policies, ACK-verified
> command injection, a web dashboard, an opt-in autonomous force-land mode, and
> opt-in evil-twin AP insertion — is built and hardware-validated. The Phase 4
> drone-side **UART/SiK backend** is built and **hardware-validated over a real
> 915 MHz SiK link** (against a software autopilot stub, 2026-06-19), and its
> **SITL firmware-logic is validated in-container** (2026-06-25). The **target
> airframe is now built** (Holybro S500 V2 + Pixhawk 6C Mini, ArduCopter 4.6.3)
> and the **SiK link to the real drone is validated** via a normal GCS
> (2026-07-02). Remaining: **chain it over the physical radios** — the ESP32 attack
> node injecting over the real SiK link into a real autopilot — gated. Phase 2's
> seven-module `iron-rain` CLI (inject, replay, GPS spoof, telemetry spoof, DoS,
> mission hijack, scan) ships alongside. See [Roadmap](#roadmap).

---

## Overview

Iron Rain makes drones fall from the sky by speaking their own protocol back
at them. Named after the orbital assault in Pierce Brown's *Red Rising*, the
project targets the **command layer** rather than the kinetic one — instead
of intercepting a drone in flight, we send it MAVLink commands it cannot
distinguish from its operator's.

The end goal is a complete kill chain: detect a hostile airframe, classify it
as MAVLink-speaking, and deliver a verified command-injection payload that
disarms it, forces a landing, or hijacks its mission — all without a kinetic
intercept and without the target's operator regaining control in time.

## Why this works

MAVLink was designed in 2009 for hobbyist telemetry. Three properties of its
default deployment make command-layer exploitation practical today:

1. **Zero authentication by default.** Any valid packet delivered to the
   autopilot's MAVLink endpoint is executed. There is no per-message identity,
   no operator binding, and no challenge–response.
2. **Signing exists but is rarely used.** MAVLink v2 added an HMAC-SHA-256
   signing extension. Real-world ArduPilot and PX4 deployments — including
   fielded military and commercial platforms — overwhelmingly ship with
   signing **disabled**, because enabling it requires shared-secret
   provisioning that most fleets don't have infrastructure for.
3. **Open ecosystem, predictable behavior.** ArduPilot and PX4 are the two
   open-source autopilot stacks underpinning the majority of long-range
   strike, reconnaissance, and ISR drones in service. Their MAVLink command
   set is public, well-documented, and consistent across vendors — which
   means an exploit developed against one airframe ports trivially.

The practical consequence: an attacker who can deliver a packet to the
autopilot — over WiFi, a captured datalink, or a co-located radio — can
disarm motors mid-flight, force a flight-mode change, hijack the active
mission, or reboot the autopilot. Iron Rain is the toolkit that turns that
capability into a repeatable, verified, and eventually autonomous workflow.

## Threat model

Iron Rain assumes the attacker has **packet-level reach** to the target's
MAVLink endpoint. That reach can come from:

- A WiFi or unencrypted RF datalink shared with the operator (Phase 3 ESP32
  device sits in this position).
- A compromised ground-control station or relay.
- A physically co-located transceiver tuned to the target's telemetry band.

Iron Rain does **not** assume:

- A pre-shared MAVLink signing key (those targets are out of scope until
  Phase 5+ key-recovery work).
- Compromise of the autopilot firmware itself.
- Any network access beyond MAVLink reachability.

## Ethics & authorized use

Iron Rain is dual-use. The same packet that disarms a hostile UAS in a
counter-drone engagement will disarm a friendly aircraft if pointed at the
wrong target — MAVLink doesn't care who's holding the radio.

**This repository exists for:**

- Counter-UAS research against hostile or simulated platforms.
- Security testing of drone systems you own or have written authorization
  to test.
- Education and training in lab environments — Damn-Vulnerable-Drone,
  SITL, CTFs, academic coursework.

**This repository is not for:**

- Targeting any aircraft you do not own or are not explicitly authorized
  to test.
- Targeting civilian, commercial, or hobbyist drones in flight.
- Any operation that could endanger people, aircraft, or property.
- Anything outside the legal framework of your jurisdiction.

The Phase 1 code in this repo only targets
[Damn-Vulnerable-Drone](https://github.com/nicholasaleks/Damn-Vulnerable-Drone) —
an intentionally vulnerable SITL target designed for exactly this kind of
work. Nothing in this phase transmits on real RF or touches a real airframe.

Operating unauthorized RF transmitters, interfering with a third party's
aircraft, or disabling a drone in flight without explicit authorization is
illegal in most jurisdictions and in many cases dangerous. Don't do it. If
you're using Iron Rain for an authorized engagement, document scope and
written authorization before you connect.

## What's in this repo today

The Phase 1 deliverable is a Python package, `ironrain`, that operates
against [Damn-Vulnerable-Drone](https://github.com/nicholasaleks/Damn-Vulnerable-Drone)
(DVD) — a Docker-based ArduCopter SITL target. Every exploit in this phase
follows the same contract:

1. Read pre-attack state from the heartbeat.
2. Send the exploit command.
3. Wait for `COMMAND_ACK` — no fire-and-forget.
4. Read post-attack state and confirm the effect actually occurred.
5. Return a typed `AttackReport` and emit a JSONL event for downstream tools.

That contract is what separates Iron Rain from a one-off pymavlink script:
every attack carries verifiable evidence of success or failure, suitable for
automated decision-making in later phases.

## Layout

```
ironrain/
├── connection.py     # MAVLink connection + heartbeat verification
├── validator.py      # Packet structure validation + assertions
├── telemetry.py      # Telemetry state reader (altitude, mode, armed status)
├── logger.py         # Structured JSON event logger
├── cli.py            # `iron-rain` Phase 2 CLI dispatcher
├── mitm.py           # frame-aware MAVLink MitM proxy (drop / rewrite / inject)
└── attacks/
    ├── inject.py            # disarm, force_land, force_rtl, reboot
    ├── replay.py            # capture + replay COMMAND_LONG traffic
    ├── gps_spoof.py         # GPS_INPUT spoofing: snap / slew / jam-then-inject
    ├── telemetry_spoof.py   # STATUSTEXT / BATTERY_STATUS spoofing
    ├── dos.py               # flood (baseline) + jam (real MitM frame-drop denial)
    ├── mission_hijack.py    # mission download / rewrite / upload
    └── scan.py              # UDP port scan for MAVLink heartbeats
tests/                # Unit tests (mocked, no DVD required)
scripts/
├── verify_dvd.py     # Smoke test: is DVD up and healthy?
└── phase1_demo.py    # Full Phase 1 demo run
docker/dvd_setup.md   # DVD Docker setup commands
iron-rain-module5/    # ESP32 bench GCS emulator (ESP-IDF firmware)
```

`iron-rain-module5/` is a bench test fixture — an ESP32 (WROOM-32) running
ESP-IDF firmware that impersonates QGroundControl/Mission Planner over MAVLink
UDP. Its job is to validate the Phase 3 MitM node against a known-good GCS
endpoint before any real airframe is involved. It is distinct from the Phase
3 MitM device itself; see [iron-rain-module5/README.md](iron-rain-module5/README.md)
for setup, build, and validation steps.

## Install

```bash
pip install -e .
```

## DVD setup

See `docker/dvd_setup.md`. Quick start:

```bash
git clone https://github.com/nicholasaleks/Damn-Vulnerable-Drone
cd Damn-Vulnerable-Drone
docker-compose up --no-wifi
```

DVD exposes MAVLink on `udp://127.0.0.1:14550`.

## Run

```bash
# Unit tests (no DVD required)
pytest tests/test_connection.py tests/test_validator.py -v

# Health check (requires DVD running)
python scripts/verify_dvd.py

# Full Phase 1 demo (requires DVD running, drone armed)
python scripts/phase1_demo.py
```

## Phase 1 attacks

| Attack       | MAV_CMD                              | Verification                      |
|--------------|--------------------------------------|-----------------------------------|
| `disarm()`   | `MAV_CMD_COMPONENT_ARM_DISARM` (400) | armed → disarmed in HEARTBEAT     |
| `force_land()` | `MAV_CMD_DO_SET_MODE` (176)        | `custom_mode == 9` (LAND)         |
| `force_rtl()`  | `MAV_CMD_DO_SET_MODE` (176)        | `custom_mode == 6` (RTL)          |
| `reboot_autopilot()` | `MAV_CMD_PREFLIGHT_REBOOT_SHUTDOWN` (246) | best-effort ACK |

Every attack returns an `AttackReport` with pre/post state, raw `MAV_RESULT`,
elapsed time, and a `verified` boolean. No fire-and-forget commands.

## Phase 2 attacks

Phase 2 ships seven attack modules behind a single `iron-rain` CLI. Each
module follows the same verification contract as Phase 1 — pre-state →
attack → post-state → typed report with a `verified` boolean.

| Module           | What it does                                      | Verification                                  |
|------------------|---------------------------------------------------|-----------------------------------------------|
| `inject`         | COMMAND_LONG injection (disarm / land / RTL / reboot) | COMMAND_ACK + heartbeat state change      |
| `replay`         | Capture COMMAND_LONG traffic, replay later        | ACK with `MAV_RESULT_ACCEPTED` after replay   |
| `gps-spoof`      | GPS_INPUT spoofing — snap / `--slew-speed` / `--jam-then-inject` | Reported lat/lon moves to target (snap ±100 m; slew/jam relocate a locked target) |
| `telemetry-spoof`| STATUSTEXT / BATTERY_STATUS injection             | Spoof packets observed echoed on the link     |
| `dos`            | `--mode flood` (app spam, baseline) or `--mode jam` (MitM frame-drop) | jam: count of victim-bound frames withheld (telemetry/command/GPS denial) |
| `mission-hijack` | Download active mission, replace, upload          | Re-downloaded mission matches uploaded plan   |
| `scan`           | UDP port probe for MAVLink heartbeats             | At least one parseable HEARTBEAT response     |

### CLI

```bash
iron-rain --conn udp:127.0.0.1:14550 inject disarm --force
iron-rain replay capture --duration 5 --out cap.jsonl
iron-rain replay replay cap.jsonl
iron-rain gps-spoof --lat 37.7749 --lon -122.4194 --duration 10
iron-rain gps-spoof --lat 37.7749 --lon -122.4194 --slew-speed 12 --duration 40   # drag a locked target
iron-rain gps-spoof --jam-then-inject --lat -34.46496 --lon 149.16523 \
  --autopilot 127.0.0.1:5760 --ground udpin:0.0.0.0:14555 --jam-seconds 12        # GPS dropout takeover
iron-rain telemetry-spoof statustext --text "BATTERY FAILSAFE"
iron-rain telemetry-spoof battery --pct 3
iron-rain dos --mode flood --duration 5 --rate 500 --type heartbeat               # baseline (shrugged off)
iron-rain dos --mode jam --drop telemetry --ground udpout:127.0.0.1:14550         # real denial (GCS blind)
iron-rain mission-hijack download
iron-rain mission-hijack land-here --lat 37.7749 --lon -122.4194
iron-rain scan --host 127.0.0.1 --ports 14550,14551,14552
```

Every subcommand prints a JSON report on stdout and exits non-zero when
the attack did not verify, so it composes cleanly into shell pipelines.
Pass `--log path.jsonl` to append every internal event to a JSONL trace.

### Running Phase 2 tests

```bash
# Unit tests — all mocked, no DVD required
pytest tests/ -v

# Skip the live bench suite explicitly:
pytest -m 'not bench'
```

### Bench integration tests

`tests/integration/` runs the attack modules end-to-end against a live
MAVLink target (DVD by default at `tcp:127.0.0.1:5760`). Tests are
auto-marked `bench` and **skip themselves** if the target is unreachable,
so an unconfigured machine still sees `pytest` go green.

```bash
# Bring DVD up first
sudo ~/Damn-Vulnerable-Drone/start.sh --mode lite --no-wifi
python3 scripts/dvd_stage.py boot       # reset + arm + takeoff

# Find flight-controller-lite's bridge IP (lite mode doesn't publish 5760
# to the host — connect via the container bridge directly)
DVD_IP=$(docker inspect -f \
  '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  flight-controller-lite)

# Run the bench suite (SITL's secondary TCP output is 5762; 5760 is taken
# by mavlink-router as input inside the container)
IRON_RAIN_BENCH_CONN=tcp:$DVD_IP:5762 pytest -m bench -v
```

What's covered: target reachability, mission upload + verify round-trip,
COMMAND_LONG → COMMAND_ACK for disarm / force-land / force-rtl,
STATUSTEXT and BATTERY_STATUS send. Stricter `verified=True` semantics
(armed → disarmed transition, mode flip) require DVD to be staged to
the right pre-state via `scripts/dvd_stage.py boot`.

The scan test (`test_scan_finds_heartbeat`) is **skipped against DVD
lite-mode** because the lite stack doesn't broadcast HEARTBEATs on a
host-reachable UDP port. Point it at Module 5 (which does) to enable it:

```bash
IRON_RAIN_BENCH_CONN=tcp:$DVD_IP:5762 \
IRON_RAIN_SCAN_HOST=<module5-ip> \
pytest -m bench -v
```

## Roadmap

- **Phase 2** — ✅ `iron-rain` Python CLI: 7 attack modules.
- **Phase 3** — ✅ ESP32 firmware ([`iron-rain-module1`](iron-rain-module1/README.md)):
  transparent MAVLink MitM with runtime attack policies, ACK-verified command
  injection, a serial + web-dashboard control surface, an opt-in autonomous
  force-land mode, and opt-in evil-twin AP insertion (the ESP32 impersonates the
  target SSID so a victim associates to it). Host-tested (`mavlink_ops` unit
  tests) and built in CI; the MitM forwarder, the operator/NRF24 standoff
  backhaul, and the evil-twin AP are all **hardware-validated** (2026-06-14).
- **Phase 4** — 🟢 Real autopilot over UART/SiK radio. The drone-side
  `drone_link_t` UDP backend has a **UART backend** alongside it (same vtable,
  `DRONE_LINK=uart`), built and CI-compiled, and **hardware-validated over a
  real 915 MHz Holybro SiK V3 pair** against a software autopilot stub
  (2026-06-19; see [`sik-bench.md`](iron-rain-module1/docs/sik-bench.md)). The
  **SITL firmware-logic is validated in-container** (2026-06-25 — the CLI injects
  verify against ArduCopter 4.6.3 SITL and the reframer recovers 99.9% of a real
  50 Hz stream; see [`sitl-bench.md`](iron-rain-module1/docs/sitl-bench.md)). The
  **target airframe is now built** — a Holybro S500 V2 + Pixhawk 6C Mini running
  ArduCopter 4.6.3 — and the **SiK link to the real drone is validated** via a
  normal GCS (Mission Planner over 915 MHz: telemetry + Motor Test, 2026-07-02;
  see [`phase4-target-drone.md`](docs/phase4-target-drone.md) and the attack-free
  build guide [`s500-ardupilot-build.md`](s500-ardupilot-build.md)).The ESP32 attack 
  node injects over the real SiK link into a real autopilot (SITL or the airframe), 
  and all commands verified run.