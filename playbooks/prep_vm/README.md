# Playbook: prep_vm

Prepares hosts in inventory group `servers` (become, role `prep_vm`).

Run from the **repository root** (`ansible.cfg` sets inventory):

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass
```

Local vault password file (gitignored `.vault_pass`):

```bash
ansible-playbook -i inventory/dev/hosts playbooks/prep_vm/prep_vm.yaml --vault-password-file .vault_pass
```

## What it expects

- OS: Debian or Ubuntu (preflight assert, tag `always` — runs with any `--tags`)
- Host var `target_partition` — LUKS target on the **non-root** disk
- Vault var `luks_passphrase` in `../../inventory/dev/group_vars/all/vault.yaml`

## Tags

| Tag | What |
| --- | --- |
| `preflight` | OS check (`always`) |
| `luks` | Encrypt inventory partition + neighbor of root |
| `cstate` | Disable C-states |
| `cpu` | `performance` governor |
| `net` | Rename active NIC to `net0` (**reboots** unless already `net0`) |
| `cpuinfo` | Print CPUs and HT/SMT (end of a full run) |

How to validate each step: [roles/prep_vm/README.md](../../roles/prep_vm/README.md).
