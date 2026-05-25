# trail-safety-mesh

Infrastructure-as-code for **Trail Safety Mesh** — a local Meshtastic-based
field coordination network for hiking, biking, small-team activity, and
future ATAK / OpenTAKServer integration.

This repository owns the **infrastructure**: it provisions and configures
the Raspberry Pi 4 field gateway and the services that run on it. Custom
runtime code (bridges, log parsers, dashboards, field APIs) lives in the
companion repository [`field-gateway-pi`](../field-gateway-pi).

## What this repo provisions

A Raspberry Pi 4 field gateway running, in layers:

1. A hardened Ubuntu 24.04 LTS base (chrony time sync, key-only SSH, UFW,
   security-only unattended upgrades, SD-wear hardening).
2. Pi-specific hardware enablement (video/GPIO/SPI/I2C groups, udev rules
   for `/dev/vcio`, intended support for the RAK6421 + RAK13300 LoRa HAT).
3. `meshtasticd` — the Linux-native Meshtastic daemon — running on the Pi.
4. (Optional) OpenTAKServer host-side prerequisites, including its own
   AMQP broker (RabbitMQ on `5672/tcp`). **There is no standalone MQTT
   broker in this build; RabbitMQ is the only broker. See ADR 0002.**

## Target

| Property             | Value                              |
| -------------------- | ---------------------------------- |
| Hardware             | Raspberry Pi 4                     |
| Radio (planned)      | RAK6421 + RAK13300 LoRa HAT        |
| OS                   | Ubuntu Server 24.04 LTS (arm64)    |
| Meshtastic daemon    | `meshtasticd` (native, not USB)    |
| MQTT broker          | Mosquitto on `1883/tcp`            |
| AMQP broker (opt.)   | RabbitMQ on `5672/tcp` (OTS only)  |

## Quick start

1. Flash Ubuntu Server 24.04 LTS (arm64) to an SD card and boot the Pi.
2. Confirm you can `ssh raspi@<pi-ip>` with a password (you'll switch to
   key-only auth on the first Ansible run).
3. On your workstation, install Ansible (`pipx install ansible` or your
   distro's package), then install the collections this playbook needs:

   ```bash
   ansible-galaxy collection install -r ansible/requirements.yml
   ```

4. Copy and edit the example inventory and group vars:

   ```bash
   cp ansible/inventory.ini.example       ansible/inventory.ini
   cp ansible/group_vars/field_gateway.yml.example \
      ansible/group_vars/field_gateway.yml
   ${EDITOR:-vi} ansible/inventory.ini ansible/group_vars/field_gateway.yml
   ```

5. Provision the gateway. For a first run we recommend limiting to the
   roles whose install paths are fully validated — `base`, `mqtt`, and
   `firewall` — and only adding `meshtasticd` and `opentakserver` once
   their TODOs are resolved on your hardware:

   ```bash
   cd ansible
   # First run -- limited scope.
   ansible-playbook -i inventory.ini site.yml \
       --tags base,mqtt,firewall \
       --ask-pass --ask-become-pass

   # Once the limited run is green and the meshtasticd / opentakserver
   # TODOs are validated, run the full playbook:
   ansible-playbook -i inventory.ini site.yml --ask-become-pass
   ```

   After the first successful run, key-only SSH is enabled; subsequent
   runs do not need `--ask-pass`.

6. Reboot once to pick up kernel/firmware updates and `fstab` changes:

   ```bash
   ssh raspi@<pi-ip> sudo reboot
   ```

7. Optional — run the OpenTAKServer installer (interactive; not Ansible-managed).
   See `docs/architecture.md` for the rationale and `roles/opentakserver`
   for the prerequisites this repo installs.

## Layout

```text
trail-safety-mesh/
├── README.md
├── ansible/
│   ├── site.yml                              # top-level playbook
│   ├── inventory.ini.example                 # copy to inventory.ini
│   ├── group_vars/
│   │   └── field_gateway.yml.example         # copy and edit
│   └── roles/
│       ├── base/                             # apt sources, packages, chrony,
│       │                                     # hostname, SSH, UFW prereqs,
│       │                                     # unattended-upgrades
│       ├── pi_hardware/                      # video/GPIO/SPI/I2C groups,
│       │                                     # /dev/vcio udev rule, fstab
│       │                                     # SD-wear options, journald caps
│       ├── meshtasticd/                      # meshtasticd install/config
│       │                                     # (TODOs noted in the role)
│       ├── mqtt/                             # Mosquitto + conf.d drop-in
│       ├── opentakserver/                    # OTS host prereqs incl. RabbitMQ
│       └── firewall/                         # UFW rules, RFC1918-scoped
├── docs/
│   ├── architecture.md
│   └── decisions/
│       ├── 0001-use-ansible-for-infrastructure.md
│       └── 0002-no-standalone-mqtt-broker.md
└── examples/
    └── README.md
```

## Companion repo

Runtime code that runs **on** the gateway — health checks, MQTT/AMQP
bridges, CoT forwarders, a field-status API — lives in
[`field-gateway-pi`](../field-gateway-pi). This repo only provisions
the OS, services, and firewall.

## License

Reference / personal-project; adapt freely.
