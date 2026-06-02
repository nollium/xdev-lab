# xdev-lab

Windows exploit development lab on QEMU/KVM. One command from exact build number to running VM with full exploit dev toolchain.

**No existing tool does this.** FLARE-VM installs tools but doesn't create VMs. Packer templates create VMs but can't target exact builds. UUP dump builds ISOs but doesn't provision anything. xdev-lab chains all three.

## Quick start

```bash
# Everything in one command: fetch WS2025 build 26100.32690 from Microsoft,
# create QEMU VM, unattended install, install exploit dev tools, promote to DC
xdev-lab create --build 26100.32690 --name ws2025 --tools exploit-dev --ad corp.local
```

## What it does

Given an exact Windows build number (e.g. `26100.32690`):

1. **Fetches the ISO** from Microsoft via [UUP dump](https://uupdump.net) — downloads packages from MS CDN and assembles a bootable ISO
2. **Creates a QEMU/KVM VM** with fully unattended Windows installation (autounattend.xml on floppy, BIOS boot, KMS keys for non-eval ISOs)
3. **Installs exploit dev tools** — crash dump config, symbol path, Sysinternals, x64dbg, Python, WinDbg
4. **Optionally promotes to Domain Controller** — for testing AD-targeting exploits

## Requirements

Linux host with:
- QEMU with KVM (`qemu-system-x86_64`, `/dev/kvm`)
- `aria2c`, `wimlib-imagex`, `cabextract`, `genisoimage` or `mkisofs` (for UUP ISO building)
- `impacket` (wmiexec for VM management via pass-the-hash)
- `nmap` (port checking during install)
- `brctl` (bridge management)
- `sudo` access (for TAP/bridge/iptables setup)

```bash
# Ubuntu/Debian
sudo apt install qemu-system-x86 aria2 wimtools cabextract genisoimage nmap bridge-utils
pip3 install impacket
```

## Commands

| Command | Description |
|---------|-------------|
| `create` | All-in-one: ISO + VM + tools + optional AD |
| `iso --build N` | Build exact-build ISO via UUP dump |
| `vm --name N --build N` | Create VM with unattended install |
| `tools --vm N` | Install exploit dev tools |
| `ad --vm N --domain D` | Promote to Domain Controller |
| `update --vm N --msu F` | Apply a Windows CU to running VM |
| `debug --vm N --mode M` | Start VM with kernel debug (gdb/serial/both) |
| `feature --vm N --enable F` | Toggle Windows features (firewall, SMBv3, KD, ASLR...) |
| `offsets --vm N --exploit E` | Calculate exploit offsets from target binaries |
| `exploit-test --vm N --poc P` | Run PoC with PCAP, crash dump, and shell monitoring |
| `snapshot N create\|restore\|list S` | QCOW2 snapshot management |
| `start N [--debug M] [--cpus N]` | Start VM with optional debug and CPU config |
| `stop\|destroy N` | VM lifecycle |
| `exec N cmd` | Run command via wmiexec (PTH) |
| `extract N remote local` | Download file from VM via SMB |
| `screenshot N` | QMP screendump |
| `list` | Show all VMs |

## Supported builds

Any Windows build indexed by UUP dump:

| Major | OS |
|-------|-----|
| `26100.x` | Windows Server 2025 / Windows 11 24H2 |
| `20348.x` | Windows Server 2022 |
| `17763.x` | Windows Server 2019 |
| `22631.x` | Windows 11 23H2 |
| `19041-19045.x` | Windows 10 (2004-22H2) |
| `18363.x` | Windows 10 1909 |
| `18362.x` | Windows 10 1903 |

## Exploit dev workflow example (SMBGhost)

```bash
# 1. Create a vulnerable Win10 1909 VM (build 18363.719 = last pre-patch)
xdev-lab create --build 18363.719 --name win10-smb --tools exploit-dev

# 2. Configure for exploit testing
xdev-lab feature --vm win10-smb --enable SingleCPU    # 1 logical CPU for reliability
xdev-lab feature --vm win10-smb --list                 # verify SMBv3 compression is on

# 3. Calculate offsets for this build
xdev-lab offsets --vm win10-smb --exploit smbghost

# 4. Start with kernel debug for crash analysis
xdev-lab start win10-smb --debug gdb --cpus 1

# 5. Run the exploit with monitoring
xdev-lab exploit-test --vm win10-smb \
    --poc ./SMBleedingGhost.py \
    --args "10.10.10.30 10.10.10.1 4444" \
    --expect shell --listen 4444

# 6. Extract binaries for offline RE
xdev-lab extract win10-smb Windows/System32/drivers/srvnet.sys ./srvnet.sys
xdev-lab extract win10-smb Windows/System32/ntoskrnl.exe ./ntoskrnl.exe
```

## How it works

- **BIOS boot** (`-machine pc`) — UEFI/OVMF has CD-ROM timeout issues with QEMU
- **FAT floppy** for autounattend.xml — Windows Setup reads `A:\autounattend.xml`
- **MBR single partition** — simplest layout, works everywhere
- **KMS generic keys** for non-evaluation ISOs (from UUP dump)
- **e1000e NIC** — no VirtIO driver needed during WinPE
- **wmiexec with pass-the-hash** — avoids `@` in password breaking CLI URI parsing
- **DHCP fallback detection** — if static IP fails, finds VM via ARP/MAC and fixes remotely
- **Firewall disabled in FirstLogonCommands** — WS2025 ignores the specialize-pass firewall component

## Network

VMs run on a `br-lab` bridge (`10.10.10.0/24`) with:
- Host at `10.10.10.1`
- dnsmasq DHCP (`10.10.10.100-200`) as fallback
- NAT to internet via iptables masquerade
- TAP interface per VM

All networking is set up automatically on first use.

## Credentials

| Field | Value |
|-------|-------|
| Username | `Administrator` |
| Password | `Lab@dmin1!` |
| NT Hash | `c07bbd6d874b375f83efdb1dd809433e` |

## License

MIT
