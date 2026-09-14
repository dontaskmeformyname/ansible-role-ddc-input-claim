# ddc_input_claim

Ansible role that claims DDC/CI monitor settings on the local host. It runs at boot and when a configured USB trigger device appears.

Install from a Git tag. Do not pin `main`.

## Requirements

- systemd and udev
- `i2c-dev` in the target kernel
- A DDC/CI-capable monitor
- Collection `community.general` (used for `community.general.modprobe`)

Supported platforms: Debian, Ubuntu, Fedora, RHEL-compatible distributions, Arch Linux, and CachyOS.

```bash
ansible-galaxy collection install -r requirements.yml
```

## Install the role

```yaml
# requirements.yml of a consuming playbook repo
roles:
  - name: ddc_input_claim
    src: https://github.com/dontaskmeformyname/ansible-role-ddc-input-claim.git
    scm: git
    version: v0.1.0

collections:
  - name: community.general
    version: ">=8.0.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

## Example

The USB and EDID values below are **examples**, not defaults. Replace them with IDs from `lsusb` and `ddcutil detect` on the target host.

```yaml
- hosts: workstations
  become: true
  roles:
    - role: ddc_input_claim
      vars:
        ddc_claim_actions:
          - { bus: "7", feature: "0x60", value: "0x0f" }
        ddc_gate_monitor:
          edid_mfg: "SAM"          # example
          edid_name: "G95NC"       # example
        ddc_gate_kvm_usb:
          vendor: "05e3"           # example Genesys hub
          product: "0610"          # example
        ddc_gate_trigger_usb:
          vendor: "046a"           # example Cherry keyboard
          product: "00ab"          # example
```

Each `ddc_claim_actions` item maps to:

```bash
ddcutil setvcp <feature> <value> --bus <bus> --noverify
```

## Safety gates

Automatic claims run only when all three Sysfs checks pass:

1. A connected DRM display matches `ddc_gate_monitor` (EDID manufacturer, optional model substring).
2. A USB hub matching `ddc_gate_kvm_usb` is present.
3. The trigger device matching `ddc_gate_trigger_usb` is a **child** of that hub.

If any check fails, the script exits 0 and does not send DDC commands. That keeps the role safe on laptops used at other desks.

Empty gate values are a configuration error, not a disabled gate.

## Variables

| Variable | Default | Meaning |
|---|---|---|
| `ddc_claim_actions` | `[]` | Required. Ordered list of `{bus, feature, value}` |
| `ddc_claim_retries` | `5` | Attempts per claim run |
| `ddc_claim_retry_delay` | `2` | Seconds between attempts |
| `ddc_claim_boot_enabled` | `true` | Enable the boot oneshot |
| `ddc_gate_monitor.edid_mfg` | `""` | Required. EDID manufacturer, e.g. `SAM` |
| `ddc_gate_monitor.edid_name` | `""` | Optional model substring |
| `ddc_gate_kvm_usb.vendor` / `product` | `""` | Required. KVM hub USB IDs |
| `ddc_gate_trigger_usb.vendor` / `product` | `""` | Required. Trigger device USB IDs |

## Manual run

```bash
sudo /usr/local/libexec/ddc-input-claim
sudo systemctl start ddc-input-claim.service

# Skip gates (tests only)
sudo /usr/local/libexec/ddc-input-claim --force

journalctl -t ddc-input-claim -e
```

## Topology changes

I2C bus numbers move after GPU, dock, or kernel changes:

```bash
sudo modprobe i2c-dev
sudo ddcutil detect
```

Update `bus` in `ddc_claim_actions`. USB IDs:

```bash
lsusb
udevadm info --attribute-walk --name=/dev/bus/usb/BBB/DDD
```

If several hubs share the same VID:PID, the parent walk is mandatory: the keyboard must sit under the configured KVM hub in Sysfs.

## Samsung Odyssey G95NC

VCP `0x60` is not a reliable per-pane PBP selector on this model. Vendor codes documented by [mwd102/g95nc-ddc](https://github.com/mwd102/g95nc-ddc) are not automated by this role until locally validated.
