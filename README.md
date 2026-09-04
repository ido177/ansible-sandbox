# ansible-sandbox2

Ansible role and playbook to prepare a Ubuntu/Debian server: LUKS on a non-root disk and on a data partition next to root, C-state off, CPU `performance` governor, rename the active NIC to `net0`, then print CPUs and Hyper-Threading / SMT.

Targeted at systemd + netplan + cloud-init (the exam Ubuntu VM). The role refuses other OS families up front.

## Layout

- [`playbooks/prep_vm/`](playbooks/prep_vm/) — playbook ([README](playbooks/prep_vm/README.md))
- [`roles/prep_vm/`](roles/prep_vm/) — role ([README](roles/prep_vm/README.md) with per-task validation)
- [`inventory/`](inventory/) — hosts and Vault (`luks_passphrase`)
- [`requirements.yml`](requirements.yml) — `community.crypto`, `community.general`
- [`ansible.cfg`](ansible.cfg) — inventory and `roles_path`

## Run

From this directory:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass
```

`--tags net` (and a full run) **reboots** the host to apply the NIC name `net0`. Have console/SSM available if SSH does not come back.

Host variables: `target_partition` (e.g. `/dev/xvdf1`). Vault: `luks_passphrase`.
