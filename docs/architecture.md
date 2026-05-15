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
membership grants and the (currently TODO) `/boot/firmware/config.txt`
overlays for the RAK HAT.

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
                              ┌──▼──┐    ┌───▼────────┐
                              │ AMQP │    │  Postgres  │
                              │ RMQ │    └────────────┘
                              └─────┘
                                 ╳     ← no bridge by default
                                 ╳        (see ADR 0002)
                              ┌──────────────────────┐
                              │   Mosquitto MQTT     │   :1883
                              │   (auth, no anon)    │
                              └──────────┬───────────┘
                                         │
                                  (MQTT pub/sub)
                                         │
                              ┌──────────▼──────────┐
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

- Provisioned by `roles/meshtasticd`. **The exact apt repo URL + package
  install path for noble/arm64 is not yet pinned**; the role ships as a
  scaffold with TODOs and does not run unverified install commands.
- Config goes in `/etc/meshtasticd/config.yaml`, deployed from a Jinja
  template only when `meshtasticd_manage_config: true`.
- Sensitive fields (channel PSK) must be supplied via Ansible Vault.

### Mosquitto

Local MQTT broker on `1883/tcp`. Authenticated; anonymous denied.

- Provisioned by `roles/mqtt`.
- Single field user (`mqtt_user`, default `fieldgw`) created by
  `mosquitto_passwd`. Password is provided via `mqtt_password` in group
  vars (preferably Vault). The role **refuses to run** if
  `mqtt_password` is empty — it will not silently downgrade to
  anonymous access.
- Logs to journald, which `roles/pi_hardware` caps at 200 MB on disk.

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
- Before running the OTS installer, **Mosquitto stays running**. The
  installer may try to bind its own broker on 1883/tcp; if it does, it
  will fail — that is the correct outcome. Mosquitto owns 1883/tcp.
  Resolve the collision by removing any listener-on-1883 directives
  from whatever `/etc/mosquitto/conf.d/*.conf` files the OTS installer
  drops, not by stopping Mosquitto. The legacy stop behaviour is
  gated behind `opentakserver_stop_mosquitto_during_install`, which
  defaults to **false** and overrides ADR 0002 if enabled.

### RabbitMQ

RabbitMQ is **only an OpenTAKServer dependency** in this architecture.

- AMQP on `5672/tcp` for OpenTAKServer / internal application messaging.
- **Not** the Meshtastic MQTT broker; that role is Mosquitto's alone.
- The `rabbitmq_mqtt` plugin **must not** be enabled, and the
  `opentakserver` role explicitly disables it on every run to keep
  1883/tcp clear for Mosquitto.
- Not exposed by the firewall role. Stays localhost / internal.
- See `docs/decisions/0002-separate-mqtt-and-rabbitmq.md`.

## Network posture

UFW rules (`roles/firewall`) are RFC1918-scoped by default. Each
inbound service is gated by an `_open` toggle in group vars:

| Port    | Service              | Toggle                        | Default |
| ------- | -------------------- | ----------------------------- | ------- |
| 22/tcp  | SSH                  | `firewall_open_ssh`           | true    |
| 1883/tcp| Mosquitto MQTT       | `firewall_open_mqtt`          | true    |
| 4403/tcp| meshtasticd TCP API  | `firewall_open_meshtasticd_api` | false |
| 8089/tcp| OTS CoT TLS          | `firewall_open_ots`           | false   |
| 8443/tcp| OTS Marti / web      | `firewall_open_ots`           | false   |
| 5672/tcp| RabbitMQ AMQP        | (never opened)                | n/a     |
| 15672/tcp| RabbitMQ mgmt UI    | (never opened)                | n/a     |

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
- `docs/decisions/0002-separate-mqtt-and-rabbitmq.md`
- Companion repo: `field-gateway-pi`
