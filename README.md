# HGX PXE Installer

Reference configs and scenario docs for bare-metal PXE reinstallation of HGX compute nodes.

---

## Architecture Overview

```
Node (UEFI PXE boot)
  │
  ├─ DHCP (aidcvm 10.2.2.171)
  │    Assigns IP + points to TFTP server + sets boot filename per node
  │
  ├─ TFTP (aidcvm 10.2.1.171)
  │    Serves UEFI bootloader (BOOTX64.EFI → grubx64.efi → grub.cfg)
  │
  └─ HTTP (aidcvm 10.2.1.171)
       Serves OS ISO + autoinstall seed (user-data / meta-data)
```

**Key hosts:**

| Host | Role | IP |
|---|---|---|
| `jump01` | Jump / gateway | 10.2.1.141 |
| `aidcvm` | DHCP + TFTP + HTTP (PXE services) | 10.2.1.171 / 10.2.2.171 |
| `mgn06` | KVM host running `aidc_rocky8` VM | — |

---

## Repo Layout

```
pxe-installer/
├── dhcpd.conf                        # DHCP host map (redacted — MACs/IPs removed)
├── tftpboot/
│   ├── grub.cfg                      # UEFI GRUB menu (per-node or global)
│   └── pxelinux.cfg/default          # Legacy BIOS fallback (not actively used)
├── autoinstall/
│   └── hgpn064/
│       ├── user-data                 # Ubuntu subiquity autoinstall config
│       └── meta-data                 # cloud-init instance identity
└── hgpn064_ubuntu2604_pxe_scenario.md  # Full install scenario log with lessons learned
```

> MAC addresses, fixed IPs, and SSH public keys are redacted in this repo.
> The live configs live on `aidcvm` at `/etc/dhcp/dhcpd.conf` and `/var/www/html/autoinstall/`.

---

## PXE Server Setup

### 1. DHCP (`/etc/dhcp/dhcpd.conf` on aidcvm)

Each node needs up to three entries:

```dhcp
# BMC (IPMI) — static IP, no PXE
host hgpnXXX-bmc  { hardware ethernet <BMC_MAC>;  fixed-address <BMC_IP>; }

# mgmt NIC (eno1) — PXE boot NIC
# Normal: no filename → boots default OS (RHEL)
host hgpnXXX-mgmt { hardware ethernet <MGMT_MAC>; fixed-address <MGMT_IP>; }

# mgmt NIC with Ubuntu override — add filename only when reinstalling with Ubuntu
host hgpnXXX-mgmt { hardware ethernet <MGMT_MAC>; fixed-address <MGMT_IP>; filename "/ubuntu-26.04-amd64/uefi/BOOTX64.EFI"; }

# Second mgmt NIC (ConnectX) — static IP only, no PXE
host hgpnXXX-mgmt1 { hardware ethernet <MGMT1_MAC>; fixed-address <MGMT1_IP>; }
```

After editing, validate and reload:

```bash
ssh jump01 'ssh root@10.2.2.171 "dhcpd -t -cf /etc/dhcp/dhcpd.conf && systemctl reload dhcpd"'
```

### 2. TFTP — Boot Files

Ubuntu PXE boot files go under `/var/lib/tftpboot/ubuntu-26.04-amd64/` on aidcvm.
Extract from the Ubuntu live ISO:

```bash
# Mount ISO and copy kernel + initrd
mount -o loop ubuntu-26.04-live-server-amd64.iso /mnt
cp /mnt/casper/vmlinuz  /var/lib/tftpboot/ubuntu-26.04-amd64/
cp /mnt/casper/initrd   /var/lib/tftpboot/ubuntu-26.04-amd64/

# Copy UEFI bootloader files
cp /mnt/EFI/boot/BOOTx64.EFI   /var/lib/tftpboot/ubuntu-26.04-amd64/uefi/BOOTX64.EFI
cp /mnt/EFI/boot/grubx64.efi   /var/lib/tftpboot/ubuntu-26.04-amd64/uefi/grubx64.efi
umount /mnt
```

Write the per-node GRUB config at `/var/lib/tftpboot/ubuntu-26.04-amd64/uefi/grub.cfg`:

```cfg
set timeout=5
set default=0

menuentry 'Autoinstall Ubuntu Server 26.04 LTS on hgpnXXX' {
  linuxefi /ubuntu-26.04-amd64/vmlinuz ip=dhcp \
    url=http://<PXE_SERVER_IP>/ubuntu/26.04/ubuntu-26.04-live-server-amd64.iso \
    autoinstall ds=nocloud-net\;s=http://<PXE_SERVER_IP>/autoinstall/hgpnXXX/ ---
  initrdefi /ubuntu-26.04-amd64/initrd
}
```

### 3. HTTP — ISO + Autoinstall Seed

Stage the ISO:

```bash
cp ubuntu-26.04-live-server-amd64.iso /var/www/html/ubuntu/26.04/
```

Create the autoinstall seed directory:

```bash
mkdir -p /var/www/html/autoinstall/hgpnXXX/
# Copy user-data and meta-data from this repo, fill in real values
cp autoinstall/hgpnXXX/user-data  /var/www/html/autoinstall/hgpnXXX/user-data
cp autoinstall/hgpnXXX/meta-data  /var/www/html/autoinstall/hgpnXXX/meta-data
```

---

## Adding a New Node

1. **Get the node's MAC addresses** — check existing `dhcpd.conf` on aidcvm or read from BMC web UI.
2. **Add DHCP entries** — BMC, mgmt (with `filename` override), mgmt1.
3. **Create autoinstall seed** — copy `autoinstall/hgpn064/` as a template, update hostname, IPs, NIC names, and storage config.
4. **Verify NIC names** — boot the live installer, drop to shell, run `ip -brief addr`. Names differ by hardware.
5. **Verify NVMe device paths** — run `lsblk` from the live installer shell. Kernel PCIe enumeration order is not guaranteed.
6. **Write GRUB config** — update `/var/lib/tftpboot/ubuntu-26.04-amd64/uefi/grub.cfg` with the new node's seed URL.
7. **Trigger PXE boot** via IPMI:

```bash
ssh jump01 'ipmitool -I lanplus -H <BMC_IP> -U admin -P "<BMC_PASSWORD>" chassis bootdev pxe options=efiboot'
ssh jump01 'ipmitool -I lanplus -H <BMC_IP> -U admin -P "<BMC_PASSWORD>" chassis power reset'
```

---

## Monitoring the Install

Watch HTTP access on aidcvm:

```bash
ssh jump01 'ssh root@10.2.2.171 "tail -f /var/log/httpd/access_log | grep --line-buffered -E \"<NODE_IP>|autoinstall|user-data\""'
```

Watch DHCP for the node's MAC:

```bash
ssh jump01 'ssh root@10.2.2.171 "journalctl -fu dhcpd --no-pager | grep --line-buffered <MGMT_MAC>"'
```

After install completes, verify:

```bash
ssh user@<BOND0_IP> 'hostname; cat /etc/os-release; ip -brief addr; cat /proc/mdstat; df -hT / /data'
```

---

## Known Pitfalls

| Issue | Cause | Fix |
|---|---|---|
| "autoinstall config did not create needed bootloader partition" | `grub_device: true` must be on the EFI **partition** object, not just the disk | Add `grub_device: true` to the partition entry in storage config |
| Wrong NVMe device paths | Kernel PCIe enumeration order varies by hardware | Always verify with `lsblk` from the live installer before writing storage config |
| Old OS still boots after install | Old OS on SATA disk (`/dev/sda`); UEFI still boots it | Add `wipe: superblock-recursive` on all non-target disks |
| SSH key not accepted after install | jump01 defaults to `id_rsa`; only ed25519 key was added | Add `id_rsa.pub` to `authorized-keys` and `late-commands` |
| Subiquity reads stale config cache | `cloud.autoinstall.yaml` overrides on restart | Place config at `/autoinstall.yaml` — highest priority in subiquity lookup chain |
