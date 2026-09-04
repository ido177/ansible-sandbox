# Role: prep_vm

Prepares a server for operation. Task files are imported from `tasks/main.yml` with tags: `luks`, `cstate`, `cpu`, `net`.

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

## C-state

Disable idle C-states on every CPU that exposes them. Two layers:

1. **Runtime** — write `1` to `/sys/devices/system/cpu/cpu*/cpuidle/stateN/disable` for `state1` and above (C0 is left alone). If cpuidle sysfs is missing (typical Xen/AWS guest), the task skips with a message and does not fail.
2. **Persistent** — drop-in `/etc/default/grub.d/99-cstate.cfg` appends `intel_idle.max_cstate=0 processor.max_cstate=0` to `GRUB_CMDLINE_LINUX`, then `update-grub`. `/etc/default/grub` is not edited. The playbook does **not** reboot here; kernel parameters apply after a later reboot.

```bash
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass --tags cstate
```

### Validate

If the hypervisor exposes cpuidle:

```bash
grep . /sys/devices/system/cpu/cpu*/cpuidle/state[1-9]*/disable
```

Each file should contain `1`.

Persistent config (before or after reboot):

```bash
cat /etc/default/grub.d/99-cstate.cfg
grep cstate /boot/grub/grub.cfg
```

After reboot:

```bash
grep -E 'intel_idle.max_cstate=0|processor.max_cstate=0' /proc/cmdline
```

On this AWS/Xen VM, runtime sysfs is likely absent; the skip message is expected. Check the drop-in and, after reboot, `/proc/cmdline`.

## CPU governor

Switch CPUs from a power-saving governor to `performance`. Two layers:

1. **Runtime** — write `performance` to each `/sys/devices/system/cpu/cpuN/cpufreq/scaling_governor` if that governor is listed in `scaling_available_governors`. If cpufreq sysfs is missing (typical Xen/AWS guest), the task skips with a message and does not fail.
2. **Persistent** — drop-in `/etc/default/grub.d/99-cpufreq.cfg` appends `cpufreq.default_governor=performance intel_pstate=performance` to `GRUB_CMDLINE_LINUX`, then `update-grub`. Separate from `99-cstate.cfg` (both append; neither overwrites the other or the cloud-init 40/50 files). The playbook does **not** reboot here.

```bash
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass --tags cpu
```

### Validate

If the hypervisor exposes cpufreq:

```bash
grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Each file should contain `performance`.

Persistent config (before or after reboot):

```bash
cat /etc/default/grub.d/99-cpufreq.cfg
ls /etc/default/grub.d/
grep default_governor /boot/grub/grub.cfg
```

`ls` should still show `40-force-partuuid.cfg`, `50-cloudimg-settings.cfg`, plus our `99-cstate.cfg` and `99-cpufreq.cfg`.

After reboot:

```bash
grep -E 'cpufreq.default_governor=performance|intel_pstate=performance' /proc/cmdline
```

On this AWS/Xen VM, runtime sysfs is likely absent; the skip message is expected. Check the drop-in and, after reboot, `/proc/cmdline`.

## Network: rename to net0

Rename the **active** NIC (the one with the default IPv4 route; on this VM `enX0`) to `net0`. The current name is taken from facts, not hardcoded.

Do not rename the interface live (`ip link` would drop SSH). The role writes persistent config and reboots:

1. `/etc/systemd/network/10-net0.link` — match by MAC, `Name=net0`.
2. `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` — stop cloud-init from recreating `enX0`.
3. `/etc/netplan/99-net0.yaml` — DHCP + the current MTU (9001 on this VM). `/etc/netplan/50-cloud-init.yaml` is removed so it cannot keep the old name.
4. Pending handlers (`update-grub`) are flushed, then `reboot` if the iface is not already `net0` or any of those files changed.
5. Facts are refreshed and the playbook **prints** net0 MAC, IPv4/IPv6, MTU, state, and the default route.

```bash
ansible-playbook playbooks/prep_vm/prep_vm.yaml --ask-vault-pass --tags net
```

This tag **reboots** the host unless it is already using `net0` with configs in place.

### Validate

```bash
ip link show net0
ip -4 addr show net0
ip route show default
ls /etc/systemd/network/10-net0.link /etc/netplan/99-net0.yaml
test ! -e /etc/netplan/50-cloud-init.yaml && echo 'cloud-init netplan removed'
```

`ip link show net0` must be UP, default route must be on `net0`, MTU must still be 9001 on this VM. The playbook log must contain the debug block with `name: net0`.
