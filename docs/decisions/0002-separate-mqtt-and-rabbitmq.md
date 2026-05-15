# ADR 0002: Keep Mosquitto MQTT and RabbitMQ AMQP separate

- **Status:** Accepted
- **Date:** 2026-05-15

## Context

The field gateway has two potential messaging brokers:

- **Mosquitto** — MQTT broker for Meshtastic traffic and any other
  field-side MQTT publishers/subscribers. Mosquitto is the canonical
  MQTT broker on Linux; Meshtastic clients (including `meshtasticd`,
  the ATAK Meshtastic plugin, and Meshtastic firmware MQTT uplink) all
  speak MQTT to it natively, on port `1883/tcp`.
- **RabbitMQ** — AMQP broker installed as a transitive dependency of
  OpenTAKServer (OTS). OTS uses RabbitMQ internally to move work
  between its `cot_parser` and the EUD handlers. It runs on
  `5672/tcp` (AMQP) and `15672/tcp` (optional management UI).

RabbitMQ technically has a plugin (`rabbitmq_mqtt`) that lets it
accept MQTT clients on `1883/tcp`. On a host running both Mosquitto
and RabbitMQ, that plugin must be disabled or the two services fight
for the same port. This was a real install-time gotcha we hit before.

The temptation is to "simplify" by collapsing the two into one broker.
Two paths suggest themselves:

- Drop Mosquitto and enable `rabbitmq_mqtt`, using RabbitMQ for
  everything.
- Drop RabbitMQ by patching OTS to use Mosquitto. (OTS does not
  support this; it's not a real option, just listed for completeness.)

We considered the first path explicitly and rejected it.

## Decision

- **Mosquitto owns MQTT on `1883/tcp`.** Meshtastic traffic and any
  custom MQTT pub/sub from `field-gateway-pi` services go through
  Mosquitto and only Mosquitto. The Mosquitto role refuses to run
  without an `mqtt_password` — it will not silently fall back to
  anonymous access.
- **RabbitMQ stays as the OTS-internal AMQP broker on `5672/tcp`.**
  It is installed by `roles/opentakserver` only when OTS is enabled.
  Its MQTT plugin (`rabbitmq_mqtt`) is **not** enabled. The
  `opentakserver` role actively disables `rabbitmq_mqtt` on every run
  so the plugin cannot grab `1883/tcp` even if a previous run or a
  third-party tool enabled it.
- **OpenTAKServer must not stop Mosquitto.** The OTS role used to
  stop+disable Mosquitto on install handoff so the OTS installer
  could bind 1883. That contradicted this ADR and is now disabled by
  default (`opentakserver_stop_mosquitto_during_install: false`). The
  legacy toggle remains as an escape hatch and emits a loud warning
  if enabled.
- **RabbitMQ is not exposed by the firewall.** `5672/tcp` and
  `15672/tcp` are not in any UFW allow rule. Treat AMQP as a private
  bus between OTS components on the same host.
- **No "blind broker mirror".** We will not configure Mosquitto to
  republish every topic into RabbitMQ (or vice-versa). The two
  brokers carry different data with different semantics; mirroring
  guarantees feedback loops and confusion.
- **A future bridge, if needed, is a deliberate translation service.**
  It would live in `field-gateway-pi/services/mqtt-rabbit-bridge/`,
  not in any Ansible role. It would translate a specific set of
  Meshtastic MQTT topics into a specific set of CoT-shaped messages
  on a specific RabbitMQ exchange, with explicit message
  transformation. It is not implemented now.

## Consequences

### Positive

- No port-1883 conflict. The OTS installer never has to fight
  Mosquitto for the MQTT port. (The handoff dance — stop Mosquitto,
  let OTS install, re-enable Mosquitto — is preserved in
  `roles/opentakserver`.)
- Two brokers with clearly different scopes:
  - Mosquitto = Meshtastic + field MQTT, exposed to the trusted LAN.
  - RabbitMQ = OTS internal, never exposed externally.
- Operationally simple to reason about: "if it's a Meshtastic packet,
  it's in Mosquitto. If it's an OTS work item, it's in RabbitMQ."
- We can disable OTS entirely without affecting Meshtastic /
  Mosquitto. We can disable Mosquitto entirely without affecting OTS.

### Negative / costs

- Two brokers to monitor, two sets of metrics, two failure domains.
  Accepted because the alternative (one broker doing both roles
  badly) is worse.
- The eventual MQTT ↔ RabbitMQ bridge has to be written as a real
  service. We are not solving "get one event from Meshtastic into OTS"
  for free.
- Operators have to remember not to enable `rabbitmq_mqtt`. Mitigated
  by a comment in `roles/opentakserver` and the firewall posture that
  would expose the misconfiguration immediately.

## Notes for any future bridge

If we ever need MQTT-side data to reach OTS:

1. **Selective topic mapping**, not "mirror everything". A bridge that
   forwards `msh/2/json/+/+` blindly into an OTS-watched RabbitMQ
   exchange is a feedback-loop hazard.
2. **Schema translation in the bridge.** Meshtastic MQTT payloads are
   not CoT. Any bridge does real parsing and re-emission, with a
   schema we own.
3. **Belongs in `field-gateway-pi/services/mqtt-rabbit-bridge/`.** Not
   in this repo. Provisioning that service from Ansible is acceptable
   later; the service code itself is not infrastructure.
