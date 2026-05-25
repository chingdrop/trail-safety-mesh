# ADR 0002: No standalone MQTT broker — RabbitMQ is the only broker

- **Status:** Accepted
- **Date:** 2026-05-24
- **Supersedes:** the earlier ADR 0002 ("Keep Mosquitto and RabbitMQ
  separate"), which is withdrawn. See "History" below.

## Context

The field gateway needs message brokering for two distinct purposes:

- **OpenTAKServer (OTS) internals.** OTS uses **RabbitMQ** to move work
  between `cot_parser` and the EUD handlers, and — once Meshtastic
  support is enabled — to carry Meshtastic `ServiceEnvelope` messages
  on the `amq.topic` exchange (routing keys like
  `opentakserver.2.e.<channel>`). RabbitMQ is installed as an OTS
  dependency. This broker is **not optional** if OTS is used.
- **General field-side pub/sub.** A standalone **Mosquitto** MQTT
  broker was originally planned for Meshtastic MQTT traffic and any
  custom `field-gateway-pi` services.

During real bring-up, two things became clear:

1. **OTS's Meshtastic ingestion is RabbitMQ-backed, not MQTT-backed.**
   The path that actually creates `euds`/`points` rows is
   ServiceEnvelope → RabbitMQ `amq.topic` → OTS MeshtasticController.
   A standalone MQTT broker is not part of that path.
2. **Running Mosquitto alongside RabbitMQ caused a real conflict.**
   RabbitMQ's `rabbitmq_mqtt` plugin and Mosquitto both bind
   `1883/tcp`; with both installed, RabbitMQ crash-looped on
   `eaddrinuse` until the plugin was disabled.

With no current consumer for a standalone broker, running one is pure
cost: a second service, a second failure domain, a second auth surface,
and a standing port conflict to remember.

## Decision

- **Do not run a standalone MQTT broker (Mosquitto) in the current
  build.** Nothing in the current system publishes or subscribes to a
  general-purpose MQTT broker.
- **RabbitMQ, installed and owned by OpenTAKServer, is the only
  broker.** It carries OTS-internal AMQP traffic and the Meshtastic
  ServiceEnvelope ingestion path. It is treated as an OTS-internal
  component, not a general field bus.
- **The `mqtt` Ansible role is retained but not run.** `mqtt_enabled`
  defaults to `false`; the `mqtt` role is excluded from the default
  `site.yml` run. The role is kept intact so that adding a standalone
  broker later is a one-line toggle, not a rewrite.
- **RabbitMQ is not exposed by the firewall.** `5672/tcp` (AMQP) and
  `15672/tcp` (management) are not in any UFW allow rule. AMQP stays
  host-internal.
- **No "blind broker mirror".** If MQTT-side data ever needs to reach
  OTS, that is a deliberate translation service (see below), not a
  topic-mirroring config.

## When to revisit

Re-introduce a standalone broker (flip `mqtt_enabled: true`, run the
`mqtt` role, open `firewall_open_mqtt`) only when there is a concrete
consumer — for example:

- A `field-gateway-pi` service that needs general-purpose pub/sub.
- A Meshtastic MQTT uplink/downlink design that genuinely needs an
  MQTT broker distinct from OTS's RabbitMQ.

Until such a consumer exists, adding the broker is speculative
infrastructure.

## Consequences

### Positive

- One broker, one failure domain, one auth surface.
- No `1883/tcp` conflict; `rabbitmq_mqtt` can be enabled or disabled
  purely on OTS's terms, with no Mosquitto to contend with.
- The Meshtastic-to-OTS path is exactly one technology (RabbitMQ),
  which matches how OTS actually ingests Meshtastic data.
- Smaller attack surface and less to monitor on a field device.

### Negative / costs

- No general-purpose MQTT bus is available for ad-hoc field-side
  publishers today. Accepted: there is no consumer for one yet.
- A future MQTT ↔ OTS bridge, if needed, must be written as a real
  translation service. It is not provided for free by broker config.

## Notes for any future bridge

If MQTT-side data ever needs to reach OTS:

1. **Selective topic mapping**, never "mirror everything" — blind
   forwarding into an OTS-watched exchange is a feedback-loop hazard.
2. **Schema translation in the bridge.** Meshtastic MQTT payloads are
   not CoT and are not OTS ServiceEnvelopes; the bridge does real
   parsing and re-emission against a schema we own.
3. **It belongs in `field-gateway-pi/services/`, not in an Ansible
   role.** Provisioning that service from Ansible later is fine; the
   service code itself is runtime, not infrastructure.

## History

The original ADR 0002 ("Keep Mosquitto MQTT and RabbitMQ AMQP
separate", dated 2026-05-15) assumed Mosquitto would be a permanent
component owning `1883/tcp`, with RabbitMQ kept strictly separate and
its MQTT plugin disabled. That assumption did not survive contact with
the real OTS Meshtastic integration, which is RabbitMQ-backed and makes
a standalone MQTT broker unnecessary for the current scope. The
two-broker ADR is withdrawn and replaced by this one. The reasoning
about a future bridge (selective mapping, schema translation, lives in
`field-gateway-pi`) carried over and still holds.
