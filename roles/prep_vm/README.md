# Role: prep_vm

Prepares a server for operation. LUKS encryption lives in `tasks/luks.yml` and is imported from `tasks/main.yml` with the `luks` tag.

## LUKS

Two independent procedures, both using LUKS2 (`community.crypto.luks_device`). No filesystem, mount, or crypttab is created.

1. **Second disk (inventory)** — encrypt the partition named in `target_partition`. If that partition does not exist yet, the role creates a GPT table and a single partition covering the whole parent disk, then formats it as LUKS. The parent disk must not hold mounted filesystems (including `/`).
2. **Partition next to root** — resolve the device mounted at `/`, read the partition table of that disk, and encrypt the physically closest neighbor (by start offset). Boot, EFI, BIOS-boot, FAT, tiny (< 100 MiB), and mounted partitions are skipped with an explicit message; the play does not fail.

Both procedures are idempotent: a second run leaves an existing LUKS container unchanged.

### Variables

| Variable | Where | Meaning |
| --- | --- | --- |
| `target_partition` | inventory host var | Partition to encrypt on the **non-root** disk, e.g. `/dev/xvdf1` or `/dev/nvme0n1p1`. Must be a partition, not a whole disk. |
| `luks_passphrase` | `inventory/group_vars/all/vault.yaml` (Vault) | Passphrase for both LUKS containers. |

### Requirements

- Collections from the repository root `requirements.yml`: `community.crypto`, `community.general`.
- `cryptsetup` and `parted` on the target (`cryptsetup` is already present on the exam VM).
- Become (the playbook sets `become: true`).

```bash
ansible-galaxy collection install -r requirements.yml
```

## Run

From the repository root. Inventory is set in `ansible.cfg`.

```bash
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass --tags luks
```

If you keep a local password file (gitignored as `.vault_pass`):

```bash
ansible-playbook playbooks/prep_vm/prep_vm.yaml --vault-password-file .vault_pass --tags luks
```

## Validate

Second-disk partition (inventory, e.g. `/dev/xvdf1`):

```bash
lsblk
sudo cryptsetup isLuks /dev/xvdf1 && echo LUKS_OK
sudo cryptsetup luksDump /dev/xvdf1
```

`isLuks` must succeed. `lsblk` should show a partition on the extra disk (not on `xvda`).

Partition next to root:

- On a typical AWS Ubuntu layout (`xvda1` = `/`, then BIOS boot / EFI `/boot`), the neighbor is **not** a data partition. The playbook must stay green and print a skip message naming the neighbor and the reason (flags, fstype, size, or mount).
- Confirm boot partitions are untouched:

```bash
lsblk /dev/xvda
findmnt / /boot /boot/efi
```

`/`, `/boot`, and `/boot/efi` must still be mounted on the same devices as before.

Re-run the playbook with `--tags luks`: the LUKS task for `target_partition` must report `changed=false`.
