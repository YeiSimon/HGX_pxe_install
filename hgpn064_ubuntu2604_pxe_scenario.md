# hgpn064 Ubuntu 26.04 PXE Install Scenario

Date: 2026-05-19

## Goal

Reinstall bare metal node `hgpn064` with Ubuntu Server 26.04 LTS by PXE/autoinstall.

## Target Node

- Hostname: `hgpn064`
- PXE/source IP: `10.2.1.64`
- Final bond0 IP: `10.2.183.64`
- Final bond1 IP: `10.2.3.64`
- BMC IP: `10.2.255.64`
- BMC user: `admin`
- BMC password: provided out-of-band, not stored here

## Jump And PXE Topology

- Jump host: `jump01`
- PXE VM host: `mgn06`
- PXE VM: `aidc_rocky8`
- PXE VM system type: libvirt/KVM/QEMU
- PXE service host: `aidcvm`
- PXE service IPs:
  - `10.2.1.171`
  - `10.2.2.171`

Discovery commands:

```bash
ssh jump01 cat /etc/hosts
ssh jump01 ssh mgn06 'virsh list --all'
ssh jump01 ssh mgn06 'virsh dumpxml aidc_rocky8'
```

`aidc_rocky8` was started with:

```bash
ssh jump01 ssh mgn06 'virsh start aidc_rocky8'
```

## Ubuntu Install Media

Requested ISO:

```text
https://ubuntu.com/download/server/thank-you?version=26.04&architecture=amd64&lts=true
```

Staged ISO on `jump01`:

```text
/data/ubuntu-26.04-live-server-amd64.iso
```

Staged ISO on PXE VM:

```text
/var/www/html/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso
```

Verified SHA256:

```text
dec49008a71f6098d0bcfc822021f4d042d5f2db279e4d75bdd981304f1ca5d9
```

HTTP URL used by installer:

```text
http://10.2.1.171/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso
```

## TFTP Boot Files

Ubuntu PXE files on `aidcvm`:

```text
/var/lib/tftpboot/ubuntu-26.04-amd64/vmlinuz
/var/lib/tftpboot/ubuntu-26.04-amd64/initrd
/var/lib/tftpboot/ubuntu-26.04-amd64/uefi/BOOTX64.EFI
/var/lib/tftpboot/ubuntu-26.04-amd64/uefi/grubx64.efi
/var/lib/tftpboot/ubuntu-26.04-amd64/uefi/grub.cfg
```

Host-specific GRUB config:

```cfg
set timeout=5
set default=0

menuentry 'Autoinstall Ubuntu Server 26.04 LTS on hgpn064' {
  linuxefi /ubuntu-26.04-amd64/vmlinuz ip=dhcp url=http://10.2.1.171/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso autoinstall ds=nocloud-net\;s=http://10.2.1.171/autoinstall/hgpn064/ ---
  initrdefi /ubuntu-26.04-amd64/initrd
}
```

## DHCP Entry

PXE NIC:

```text
eno1 MAC: 10:7c:61:54:e0:58
```

DHCP host override in `/etc/dhcp/dhcpd.conf` on `aidcvm`:

```dhcp
host hgpn064-mgmt {
  hardware ethernet 10:7c:61:54:e0:58;
  fixed-address 10.2.1.64;
  filename "/ubuntu-26.04-amd64/uefi/BOOTX64.EFI";
}
```

Validate DHCP config:

```bash
ssh jump01 'ssh root@10.2.2.171 "dhcpd -t -cf /etc/dhcp/dhcpd.conf"'
```

Services:

```bash
ssh jump01 'ssh root@10.2.2.171 "systemctl status dhcpd --no-pager"'
ssh jump01 'ssh root@10.2.2.171 "systemctl status tftp.socket --no-pager"'
```

## Autoinstall Seed

NoCloud seed directory on `aidcvm`:

```text
/var/www/html/autoinstall/hgpn064/
```

Files:

```text
/var/www/html/autoinstall/hgpn064/meta-data
/var/www/html/autoinstall/hgpn064/user-data
```

Seed URL:

```text
http://10.2.1.171/autoinstall/hgpn064/
```

`meta-data`:

```yaml
instance-id: hgpn064-ubuntu-2604
local-hostname: hgpn064
```

Installer identity:

```yaml
hostname: hgpn064
username: user
password: SHA-512 hash for the requested password
```

Do not store the plaintext OS password in this file.

## Network Plan

`eno1np0` (PXE/mgmt NIC):

```text
address: 10.2.1.64/24
```

`bond0`:

```text
slaves: ens255f0np0, ens3f0np0
mode: 802.3ad
hash policy: layer3+4
address: 10.2.183.64/24
gateway: 10.2.183.254
DNS: 10.2.2.141, 10.2.2.142
```

`bond1`:

```text
slaves: ens255f1np1, ens3f1np1
mode: 802.3ad
hash policy: layer3+4
address: 10.2.3.64/24
```

Note: NIC names verified from `ip -brief addr` on live installer. Initial assumptions used wrong names.

## Storage Plan

Boot/root disks (verified from live installer `lsblk`):

```text
/dev/nvme6n1  1.7T  ← grub_device, EFI + RAID1 root
/dev/nvme8n1  1.7T  ← EFI + RAID1 root
```

Root RAID:

```text
md0
RAID1
filesystem: xfs
mountpoint: /
```

Data disks (verified from live installer `lsblk`):

```text
/dev/nvme0n1  3.5T
/dev/nvme1n1  3.5T
/dev/nvme2n1  3.5T
/dev/nvme3n1  3.5T
/dev/nvme4n1  3.5T
/dev/nvme5n1  3.5T
/dev/nvme7n1  3.5T
/dev/nvme9n1  3.5T
```

Data RAID:

```text
md1
RAID6
filesystem: xfs
mountpoint: /data
```

Also wiped during install (contained old OS):

```text
/dev/sda  57.3G  ← old OS root (ext4), wiped
/dev/sdb  28.7G  ← wiped
```

Important: NVMe device names are assigned by kernel PCIe enumeration order and
may differ from expectations. Always verify with `lsblk` from the live installer
before writing the storage config.

Packages requested in autoinstall:

```text
mdadm
xfsprogs
```

## Start PXE Install

The OS-level method used first:

```bash
ssh hgpn064 "printf '<OS_PASSWORD>\n' | sudo -S efibootmgr -n 0003 && printf '<OS_PASSWORD>\n' | sudo -S systemctl reboot"
```

`Boot0003` was selected because it matched:

```text
UEFI PXE IPv4 Intel X710
MAC: 10:7c:61:54:e0:58
```

BMC/IPMI note:

```bash
ssh jump01 'ipmitool -I lanplus -H 10.2.255.64 -U admin -P "<BMC_PASSWORD>" chassis status'
```

IPMI RMCP+ works with the admin account. Use this to force PXE boot:

```bash
ssh jump01 'ipmitool -I lanplus -H 10.2.255.64 -U admin -P "<BMC_PASSWORD>" chassis bootdev pxe options=efiboot'
ssh jump01 'ipmitool -I lanplus -H 10.2.255.64 -U admin -P "<BMC_PASSWORD>" chassis power reset'
```

## Observed Successful PXE Handoff

DHCP/TFTP logs on `aidcvm` showed the target booted from PXE:

```text
DHCPDISCOVER from 10:7c:61:54:e0:58 via eth1
DHCPOFFER on 10.2.1.64 to 10:7c:61:54:e0:58 via eth1
DHCPREQUEST for 10.2.1.64 (10.2.1.171) from 10:7c:61:54:e0:58 via eth1
DHCPACK on 10.2.1.64 to 10:7c:61:54:e0:58 via eth1
Client 10.2.1.64 finished /ubuntu-26.04-amd64/uefi/BOOTX64.EFI
Client 10.2.1.64 finished /ubuntu-26.04-amd64/uefi/grubx64.efi
Client 10.2.1.64 finished /ubuntu-26.04-amd64/uefi/grub.cfg
Client 10.2.1.64 finished /ubuntu-26.04-amd64/vmlinuz
Client 10.2.1.64 finished /ubuntu-26.04-amd64/initrd
```

HTTP logs showed the installer fetched the ISO and autoinstall seed:

```text
10.2.1.64 GET /ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso 200
10.2.1.64 GET /autoinstall/hgpn064/meta-data 200
10.2.1.64 GET /autoinstall/hgpn064/user-data 200
```

Expected 404s:

```text
/autoinstall/hgpn064/network-config
/autoinstall/hgpn064/vendor-data
```

Those are normal for this NoCloud seed layout because network config is embedded in `user-data` and no vendor-data file was provided.

## Current Last Known State

Installation completed successfully on 2026-05-20.

- hostname: `hgpn064`
- OS: Ubuntu 26.04 LTS, kernel `7.0.0-14-generic`
- `eno1np0`: `10.2.1.64/24`
- `bond0`: `10.2.183.64/24`
- `bond1`: `10.2.3.64/24`
- `md0`: RAID1 on `nvme6n1p2` + `nvme8n1p2` → xfs → `/`
- `md1`: RAID6 on 8x 3.5T NVMe → xfs → `/data`
- SSH access: `ssh user@10.2.183.64` works from jump01 (no `-i` flag needed)
- Root account locked (`passwd -l root`)

## Known Issues and Fixes Applied

| Issue | Root Cause | Fix |
|---|---|---|
| Bootloader partition error | `grub_device: true` was on disk only; subiquity UEFI check requires it on the EFI **partition** | Added `grub_device: true` to `nvme6-efi` partition |
| Wrong NVMe paths | Kernel PCIe enumeration assigned different numbers than expected | Verified with `lsblk` from live installer; boot=nvme6/nvme8, data=nvme0-5,7,9 |
| Old kernel panic on boot | Old OS was on `/dev/sda` (SATA); UEFI was still booting it | Added `wipe: superblock-recursive` on sda/sdb in storage config |
| SSH key not working after install | jump01 defaults to `id_rsa` but only `DFT` (ed25519) was added | Added `id_rsa.pub` (ASUS key) to both `ssh.authorized-keys` and `late-commands` |
| Subiquity config cache | `cloud.autoinstall.yaml` kept reverting to old config on restart | Used `/autoinstall.yaml` which takes priority over cloud cache |

## Monitor Commands

Watch HTTP seed/ISO access:

```bash
ssh jump01 'ssh root@10.2.2.171 "tail -f /var/log/httpd/access_log | grep --line-buffered -E '\''10\\.2\\.1\\.64|10\\.2\\.183\\.64|autoinstall|user-data|meta-data|ubuntu-26\\.04'\''"'
```

Watch DHCP for the target MAC:

```bash
ssh jump01 'ssh root@10.2.2.171 "journalctl -fu dhcpd --no-pager | grep --line-buffered -E '\''10:7c:61:54:e0:58|10.2.1.64|DHCPOFFER|DHCPACK'\''"'
```

Check SSH ports:

```bash
ssh jump01 'for h in 10.2.1.64 10.2.183.64 10.2.3.64; do echo $h; nc -vz -w 2 $h 22; done'
```

Detect reboot from live installer to final installed OS by watching host key change or SSH availability:

```bash
ssh jump01 'for h in 10.2.1.64 10.2.183.64 10.2.3.64; do echo $h; ssh-keyscan -T 2 $h 2>/dev/null | grep ssh-ed25519; done'
```

After final reboot, verify:

```bash
ssh user@10.2.183.64 'hostname; cat /etc/os-release; ip -brief addr; cat /proc/mdstat; lsblk -f; df -hT / /data'
```

Expected final checks:

```text
hostname: hgpn064
Ubuntu: 26.04 LTS
bond0: 10.2.183.64/24
bond1: 10.2.3.64/24
eno1: 10.2.1.64/24
/: xfs on md0 RAID1
/data: xfs on md1 RAID6
```
