# Architecture

This document describes the runtime layout of a Trail Safety Mesh field
gateway and why the pieces are arranged the way they are.

## Scope

A field gateway is a single Raspberry Pi 4 running Ubuntu Server 24.04
LTS (arm64). It serves three purposes:

1. **Mesh radio node.** Run `meshtasticd` so the Pi participates in the
   Meshtastic LoRa mesh natively, with no separate USB device required.
2. **MQTT broker.** Run Mosquitto as the local MQTT broker for
   Meshtastic packets and any other field-side MQTT traffic on the
   trusted network.
3. **(Optional) TAK server host.** Run OpenTAKServer for ATAK / iTAK
   client integration when the deployment calls for it.

The two repos that ship this system have separate responsibilities:

- `trail-safety-mesh` (this repo) — Ansible playbook + roles that
  provision the gateway: OS hardening, packages, services, firewall.
- `field-gateway-pi` — runtime code that runs **on** the gateway:
  health checks, diagnostics, future bridge services, future
  field-status API.

## Hardware

| Component         | Notes                                                     |
| ----------------- | --------------------------------------------------------- |
| Raspberry Pi 4    | 4 GB or 8 GB. 8 GB recommended once OTS is enabled.       |
| SD card           | A2-class endurance preferred. SD-wear hardening is on.    |
| LoRa HAT (planned)| RAK6421 carrier + RAK13300 (SX1262) module.               |
| Power             | Official Pi 4 supply or vehicle DC-DC; UPS HAT optional.  |

`meshtasticd_rak_hat_present` in group vars gates the SPI/GPIO/I2C group
membership grants and the `/boot/firmware/config.txt` bus-enable lines
for the RAK HAT (`roles/pi_hardware/tasks/radio.yml`).

## Software layers

```
                                 ATAK / iTAK clients
                                         │
                                   (CoT / TLS)
                                         │
                              ┌──────────▼──────────┐
                              │  OpenTAKServer      │   optional, opt-in
                              │  (Marti, EUD, CoT)  │
                              └──┬───────────┬──────┘
                                 │           │
                          (RabbitMQ 5672)    │
                                 │           │
                              ┌──▼───────┐ ┌─▼──────────┐
                              │ RabbitMQ │ │  Postgres  │
                              │  AMQP +  │ └────────────┘
                              │  MQTT-   │
                              │  ingest  │   the only broker
                              └──▲───────┘
                                 │
                  Meshtastic ServiceEnvelope ingestion
                  (amq.topic / opentakserver.2.e.<channel>)
                                 │
                              ┌──┴──────────────────┐
                              │     meshtasticd     │   Linux-native
                              │  (RAK6421+RAK13300) │   Meshtastic daemon
                              └──────────┬──────────┘
                                         │
                                    (LoRa radio)
                                         │
                                  Field Meshtastic
                                  nodes (phones,
                                  handheld units)
```

## Service responsibilities

### meshtasticd

The Linux-native Meshtastic daemon. Replaces the previous design where
a USB-attached RAK4631 acted as the Pi's radio. With meshtasticd plus
the RAK6421+RAK13300 HAT, the Pi *is* the radio node.

- Provisioned by `roles/meshtasticd`. Installed from the Meshtastic
  Ubuntu PPA `ppa:meshtastic/beta` (the more conservative of the two
  PPA channels — chosen so the radio daemon is not exposed to
  alpha-channel churn).
- Hardware config is a drop-in under `/etc/meshtasticd/config.d/`.
  meshtasticd merges every file in that directory; the role templates
  one file for the RAK6421+RAK13300 slot-1 wiring (SX1262, with the
  IRQ/Reset/Busy GPIO mapping captured from a working unit). The
  GPIO/SPI values live in `roles/meshtasticd/defaults/main.yml`.
- Region, channels, and identity are **not** managed by this role.
  meshtasticd or a Meshtastic client sets those. LoRa region is
  legally required and is set on the device, not in Ansible.

### OpenTAKServer (optional)

Off by default. Set `opentakserver_enabled: true` in group vars to
install the host-side prerequisites (Python venv tooling, build chain,
nginx, postgresql, rabbitmq-server).

- The OTS application itself is **not** installed by Ansible. The
  upstream installer is interactive (CA fields, admin password,
  MediaMTX yes/no) and downloads code from i.opentakserver.io. Run it
  manually after the playbook finishes:
  ```
  curl -s -L https://i.opentakserver.io/ubuntu_installer | bash -
  ```
- The OTS installer historically reports "Installed!" even on
  catastrophic failure. Verify after each run with:
  ```
  systemctl is-active opentakserver cot_parser eud_handler eud_handler_ssl mediamtx
  ```
- This build runs no standalone MQTT broker, so the OTS installer is
  free to configure RabbitMQ — including its `rabbitmq_mqtt` plugin —
  however it needs. There is nothing for it to collide with on
  `1883/tcp`.

### RabbitMQ

RabbitMQ is installed as an OpenTAKServer dependency, and in this
build it is the **only** message broker.

- AMQP on `5672/tcp` for OTS-internal application messaging.
- Also carries the Meshtastic ingestion path when OTS Meshtastic
  support is enabled: `ServiceEnvelope` messages on the `amq.topic`
  exchange, routing keys `opentakserver.2.e.<channel>`.
- Whether the `rabbitmq_mqtt` plugin is enabled is left to OTS; this
  repo does not force it on or off. With no Mosquitto present there is
  no `1883/tcp` contention either way.
- Not exposed by the firewall role. Stays localhost / host-internal.
- See `docs/decisions/0002-no-standalone-mqtt-broker.md`.

### MQTT (Mosquitto) — not deployed

There is no standalone MQTT broker in the current build. The `mqtt`
role still exists for a possible future need (a custom `field-gateway-pi`
pub/sub service), but `mqtt_enabled` defaults to `false` and the role
is not in the default `site.yml` run. See ADR 0002.

## Network posture

UFW rules (`roles/firewall`) are RFC1918-scoped by default. Each
inbound service is gated by an `_open` toggle in group vars:

| Port    | Service              | Toggle                          | Default |
| ------- | -------------------- | ------------------------------- | ------- |
| 22/tcp  | SSH                  | `firewall_open_ssh`             | true    |
| 443/tcp | OTS HTTPS web/API    | `firewall_open_ots`             | false   |
| 8443/tcp| OTS Marti / TAK      | `firewall_open_ots`             | false   |
| 8446/tcp| OTS cert enrollment  | `firewall_open_ots`             | false   |
| 8089/tcp| OTS CoT TLS          | `firewall_open_ots`             | false   |
| 4403/tcp| meshtasticd TCP API  | `firewall_open_meshtasticd_api` | false   |
| 1883/tcp| (future Mosquitto)   | `firewall_open_mqtt`            | false   |
| 5672/tcp| RabbitMQ AMQP        | (never opened)                  | n/a     |
| 15672/tcp| RabbitMQ mgmt UI    | (never opened)                  | n/a     |

OTS certificate enrollment on `8446/tcp` is easy to miss: ATAK tablet
registration fails outright if it is blocked. All four OTS ports are
gated by the single `firewall_open_ots` toggle.

## Custom runtime code lives elsewhere

This repo does **not** contain custom Python services, MQTT-to-CoT
bridges, log parsers, or dashboards. Those live in `field-gateway-pi`.
Anything that would have been "a small extra script the gateway runs"
belongs there, not in an Ansible role here.

A future MQTT ↔ RabbitMQ bridge, if one is ever built, is a deliberate
selective-translation service that lives in
`field-gateway-pi/services/mqtt-rabbit-bridge`. See ADR 0002.

## See also

- `docs/decisions/0001-use-ansible-for-infrastructure.md`
- `docs/decisions/0002-no-standalone-mqtt-broker.md`
- Companion repo: `field-gateway-pi`
