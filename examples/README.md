# examples/

Reference snippets and end-to-end usage examples that don't belong in
the playbook itself. Intended contents (add as needed):

- A fully filled-out `group_vars/field_gateway.yml` for a typical
  Mosquitto-only deployment (no OTS).
- A fully filled-out `group_vars/field_gateway.yml` for a deployment
  that also enables OpenTAKServer.
- An `inventory.ini` with multiple gateways and a non-default user.

Keep anything with real secrets out of this directory. Use
`<REDACTED>` placeholders for passwords, channel PSKs, and SSH keys.
