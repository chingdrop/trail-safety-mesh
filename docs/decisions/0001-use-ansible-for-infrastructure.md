# ADR 0001: Use Ansible as the source of truth for infrastructure

- **Status:** Accepted
- **Date:** 2026-05-15
- **Supersedes:** the previous Bash-based `bootstrap.sh` + numbered
  `scripts/NN-*.sh` flow in the old `field-gateway-pi` repo.

## Context

The original field-gateway provisioning was a Bash bootstrap. A driver
script (`bootstrap.sh`) ran a numbered sequence of phase scripts:

```
scripts/00-apt-sources.sh
scripts/01-os-update.sh
scripts/02-time-chrony.sh
...
scripts/11-ots-prereqs.sh
```

Each phase script was idempotent on its own, and a shared `lib/common.sh`
provided logging, package-install helpers, and an `apt_install_if_missing`
function. The approach worked: it correctly built field gateways and was
runnable phase-by-phase.

As the project grew, three problems became sharper:

1. **Provisioning logic was scattered across Bash.** Idempotency was
   hand-rolled for each concern (a custom `write_file_idempotent`, a
   custom `append_line_idempotent`, ad-hoc `grep` guards). Each new
   service added another hand-rolled idempotency check. Reviewing
   correctness required reading shell.
2. **No structured separation of provisioning from runtime code.**
   The same repo housed both phase scripts and intended-to-be-runtime
   helpers. As the runtime side grows (bridges, dashboards, field APIs),
   keeping it in the same repo as the OS provisioning blurs ownership.
3. **Multi-host inventory was awkward.** Bash scripts run on one host
   at a time. A future fleet (multiple gateways at different sites)
   needs grouped configuration, host-specific overrides, and parallel
   execution. Bash is the wrong tool for that.

## Decision

- **Ansible is the source of truth for provisioning.** A new repo,
  `trail-safety-mesh`, contains the playbook (`ansible/site.yml`) and
  roles (`base`, `pi_hardware`, `meshtasticd`, `mqtt`, `opentakserver`,
  `firewall`).
- **Bash installers are deprecated for provisioning.** The numbered
  phase scripts from the old repo are not carried forward as
  provisioning artifacts. Their *logic* is converted role-by-role into
  Ansible tasks. The originals are not preserved in the new repos.
- **Bash remains acceptable for runtime helpers** (health checks,
  diagnostics, MQTT test publish/subscribe, log collection). These
  live in `field-gateway-pi/scripts/`, not in this repo.
- **No host-specific values in committed files.** Inventory and group
  vars are committed only as `.example` files; the real files are
  gitignored and supplied per deployment.

## Consequences

### Positive

- Idempotency comes from Ansible modules (`apt`, `systemd`, `copy`,
  `template`, `lineinfile`, `mount`, `ufw`) rather than hand-rolled
  shell guards. Less surface area to get wrong.
- Configuration becomes data: `group_vars/field_gateway.yml` documents
  exactly what a deployment looks like.
- Runs are reportable. Ansible's `changed`/`ok`/`skipped` summary
  replaces ad-hoc `log_ok` / `log_skip` color output.
- Future additions (a second gateway, a control node) plug into the
  same inventory.
- Sensitive material (passwords, channel PSKs) routes cleanly through
  Ansible Vault.

### Negative / costs

- Ansible is a new dependency on the operator's workstation. For a
  single-Pi homelab deployment that's overhead. We accept it because
  this project will not stay a single-Pi homelab indefinitely.
- Some operations are awkward in pure Ansible: notably, `mosquitto_passwd`
  has no native idempotent mode. We accept a `changed_when: true` shell
  task with `no_log: true` in `roles/mqtt`.
- The Pi `/dev/vcio` udev rule and `mosquitto_passwd` invocation still
  use shell-like patterns under the hood. That's fine — Ansible-as-orchestration
  doesn't forbid shelling out, it just makes it the exception.

### Migration notes

- Anyone running the old `./bootstrap.sh` flow should treat that repo
  as archived and switch to running `ansible-playbook -i inventory.ini
  site.yml`. Idempotency makes the switch safe on already-provisioned
  hosts: running Ansible against a host the old Bash flow already
  configured will mostly report `ok`.
- The pre-OTS-install Mosquitto stop is now in `roles/opentakserver`
  rather than `scripts/11-ots-prereqs.sh`. Behavior is the same.
- The `sshd_config.d/01-local-hardening.conf` first-wins precedence
  trick (filename starts with `01-`, not `99-`) is preserved verbatim
  in `roles/base/tasks/ssh_hardening.yml`. The comment explaining why
  is preserved with it.
