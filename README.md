# ddc_input_claim

Ansible-Role für einen host-lokalen DDC/CI-Claim: Ein Rechner setzt Monitor-VCP-Werte beim Boot sowie beim Erscheinen eines USB-Triggergeräts.

## Sicherheitsmodell

Vor jedem automatischen Claim prüft das installierte Gate-Skript ausschließlich über Sysfs:

1. Ein passender Monitor ist per DRM verbunden (EDID-Hersteller und optional Modell-Substring).
2. Ein passender USB-KVM-Hub ist vorhanden.
3. Das USB-Triggergerät ist ein Nachkomme dieses KVM-Hubs.

Fehlt eine Bedingung, endet das Script mit Exit-Code 0 und führt keinen DDC-Befehl aus. Das macht die Role für mobile Hosts geeignet: An einem anderen Arbeitsplatz entstehen weder lange Retries noch Änderungen am dortigen Monitor.

## Unterstützte Plattformen

Ubuntu, Debian, Fedora, RHEL-kompatible Distributionen sowie Arch Linux/CachyOS. Die Role verwendet `ansible.builtin.package`, systemd und udev.

## Voraussetzungen

- systemd/udev
- `i2c-dev` im Zielkernel
- ein DDC/CI-fähiger Monitor
- Ansible Collection `community.general` ist nicht erforderlich

## Beispiel

```yaml
- hosts: clients
  become: true
  roles:
    - role: ddc_input_claim
      vars:
        ddc_claim_actions:
          - { bus: "7", feature: "0x60", value: "0x0f" }
        ddc_gate_monitor:
          edid_mfg: "SAM"
          edid_name: "G95NC"
        ddc_gate_kvm_usb:
          vendor: "05e3"
          product: "0610"
        ddc_gate_trigger_usb:
          vendor: "046a"
          product: "00ab"
```

`ddc_claim_actions` ist bewusst generisch. Eine Aktion entspricht einem `ddcutil setvcp <feature> <value> --bus <bus> --noverify`.

## Variablen

| Variable | Typ | Standard | Bedeutung |
|---|---:|---|---|
| `ddc_claim_actions` | Liste | `[]` | Pflicht. DDC-Operationen in Ausführungsreihenfolge |
| `ddc_claim_retries` | Integer | `5` | Anzahl Versuche pro Claim-Lauf |
| `ddc_claim_retry_delay` | Integer | `2` | Pause zwischen Versuchen in Sekunden |
| `ddc_claim_boot_enabled` | Boolean | `true` | Claim-Oneshot beim Boot aktivieren |
| `ddc_gate_monitor.edid_mfg` | String | `""` | EDID-Hersteller, z. B. `SAM`; leer deaktiviert den Teilcheck |
| `ddc_gate_monitor.edid_name` | String | `""` | optionaler Modell-Substring; leer deaktiviert ihn |
| `ddc_gate_kvm_usb.vendor/product` | String | `""` | KVM-Hub USB-VID/PID; beide nötig |
| `ddc_gate_trigger_usb.vendor/product` | String | `""` | Triggergerät USB-VID/PID; beide nötig |

## Manuelle Ausführung

```bash
# Gleiche Gates wie Boot und udev
sudo /usr/local/libexec/ddc-input-claim
sudo systemctl start ddc-input-claim.service

# Gates absichtlich übergehen; nur für Tests
sudo /usr/local/libexec/ddc-input-claim --force

journalctl -t ddc-input-claim -e
```

## Anpassungen nach Topologieänderungen

I²C-Busnummern können sich nach Dock-, GPU- oder Kerneländerungen ändern. Neu ermitteln:

```bash
sudo modprobe i2c-dev
sudo ddcutil detect
```

Danach `bus` in `ddc_claim_actions` anpassen. USB-IDs bestimmen:

```bash
lsusb
# Detail inklusive Parentpfad:
udevadm info --attribute-walk --name=/dev/bus/usb/BBB/DDD
```

Wenn mehrere baugleiche Hubs existieren, ist der Parent-Check entscheidend: Die Tastatur muss im USB-Sysfs unter dem konfigurierten KVM-Hub liegen. Der Gate-Code prüft genau diese Vorfahrenbeziehung und nicht nur eine globale VID:PID-Suche.

## Samsung G95NC

Der Standard-VCP-Code `0x60` verhält sich beim G95NC nicht MCCS-konform und ist nicht ausreichend für pane-spezifische PBP-Steuerung. Das Projekt [mwd102/g95nc-ddc](https://github.com/mwd102/g95nc-ddc) dokumentiert proprietäre Codes für Layout und Pane-Quellen. Diese Role installiert oder verwendet dieses Backend bewusst noch nicht, bis die konkreten Befehle und Wirkungen lokal validiert sind.
