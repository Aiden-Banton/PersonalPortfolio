# EVE-NG Setup Guide

EVE-NG Community Edition as a VM on Proxmox Node 1 (the M910q), and how to get device images imported so they actually appear in the node picker. This is the expanded version of step 6.1 in `DEPLOYMENT-GUIDE.md`.

The M910q is dedicated to this VM specifically because EVE-NG is the one workload in the cluster that spikes hard and unpredictably — a six-node topology booting at once will take everything it's given. Keeping it off Nodes 2 and 3 is what stops a lab run from disturbing DNS, Jellyfin, or Home Assistant. See `SERVICES.md` for the full node split.

Which images are installed and which labs are waiting on one is tracked separately in [`../Labs/EVE-NG-Images/README.md`](../Labs/EVE-NG-Images/README.md). This guide is the *how*; that file is the *what*.

## Community vs Professional

Community Edition is free and covers everything the CCNA and JNCIA labs need. The paid Professional edition adds multi-server clustering, live topology editing, and a few UI conveniences. CE's practical limits: no clustering, and a documented 16-core-per-VM ceiling that Professional lifts. On a 4-core M910q that ceiling is irrelevant.

This is the bare-metal ISO install run inside a Proxmox VM — the same installer EVE-NG ships for dedicated hardware, pointed at a VM instead. It runs well. Worth knowing that EVE-NG's officially supported list is VMware, ESXi, and physical hardware, so Proxmox questions on their forum tend to get "try it on a supported hypervisor" as a first response. That's a support-channel limitation, not a functional one.

## 1. Before creating the VM: nested virtualization

EVE-NG runs QEMU/KVM *inside* a VM, so the host has to expose virtualization extensions to the guest. This is the single most common cause of "EVE-NG installs fine but no node will boot."

**On the Proxmox host**, confirm nested virtualization is on:

```
cat /sys/module/kvm_intel/parameters/nested
```

`Y` means enabled. If it returns `N`:

```
echo "options kvm-intel nested=Y" > /etc/modprobe.d/kvm-intel.conf
modprobe -r kvm_intel && modprobe kvm_intel
```

Make it survive reboots with `update-initramfs -u`, then reboot the node. Also confirm VT-x/VT-d are enabled in the M910q's BIOS — that's Section 1 of `DEPLOYMENT-GUIDE.md`, and it's a prerequisite for this working at all.

## 2. Create the VM

| Setting | Value | Why |
|---|---|---|
| **CPU type** | `host` | **Non-negotiable.** The default `kvm64` hides the virtualization flags from the guest and nested KVM will not work. This is the setting people miss |
| Cores | 4 (all of them) | The M910q's full allocation; EVE-NG is the only thing on this node |
| Sockets | 1 | |
| RAM | 24576 MB (24 GB) | Leaves ~8 GB for Proxmox host overhead out of 32 GB. Ballooning off — EVE-NG manages its own guest memory and ballooning interferes |
| Disk | 250 GB+, `VirtIO Block` | On the M910q's dedicated 1 TB drive, not the OS drive (`SERVICES.md` storage standard). Images are large and lab node disks are thin-provisioned on top of them |
| Disk cache | `Write back` | Noticeably faster node boots; acceptable here because the VM is backed up and rebuildable |
| Network | `VirtIO (paravirtualized)`, VLAN 20 | Same segment as the rest of the cluster |
| BIOS | SeaBIOS (default) | No reason to use OVMF/UEFI |
| Start at boot | Yes | |

Size the disk generously. Running out of disk on EVE-NG usually means rebuilding rather than repairing, and a single QEMU image can be several GB before any lab node is created. EVE-NG publishes a resource calculator on their download page — whatever it says, add headroom.

## 3. Install

1. Download the EVE-NG CE ISO from the [official download page](https://www.eve-ng.net/index.php/download/) (free account required) and upload it to Proxmox ISO storage.
2. Boot the VM from it. Pick language and keyboard layout — that's all the installer asks.
3. Confirm the destructive-install warning. The system reboots on its own.
4. **At the first login prompt after that reboot, do not log in.** A second install stage runs automatically and needs a few minutes. Logging in interrupts it. Wait for the reboot.
5. Log in as `root` / `eve`. The first-boot wizard runs:

| Prompt | Value for this lab |
|---|---|
| New root password | Set a real one; store it in the password manager, not in this repo |
| Hostname | `eve` |
| DNS domain | `lab.local` — matches the pfSense host overrides in `PFSENSE-SERVICES.md` |
| DHCP or static | **static** |
| IP address | `10.10.20.50` (per `ADDRESSING.md`) |
| Netmask | `255.255.255.0` |
| Gateway | `10.10.20.1` |
| Primary DNS | `10.10.20.1` — pfSense's resolver, not the Pi-hole VIP. VLAN 20 uses the gateway by design (`ADDRESSING.md`), so EVE-NG doesn't depend on DNS running as a guest elsewhere in the cluster |
| Secondary DNS / NTP | Enter through, or set NTP to `10.10.20.1` once `PFSENSE-SERVICES.md` is done |
| Proxy | Direct connection |

Another reboot follows.

6. Update, since the ISO is usually behind:
```
apt update && apt upgrade
```
When asked about replacing a config file that was modified, keep the existing one. Reboot after.

7. Verify the package is present and note the version:
```
dpkg -l eve-ng
```

8. Browse to `http://10.10.20.50` and log in with `admin` / `eve`. **Change that password immediately** — the web UI is reachable from TRUSTED and MEDIA per the firewall rules in `PFSENSE-SETUP.md`, so the default credentials are exposed to every device in the house.

## 4. Verify nested virtualization actually reached the guest

Do this *before* importing images. If it fails, no QEMU node will ever boot and you'll waste an afternoon blaming the image.

```
egrep -c '(vmx|svm)' /proc/cpuinfo     # should be > 0
apt install cpu-checker && kvm-ok      # "KVM acceleration can be used"
```

Zero means the VM's CPU type isn't `host` — go back to Section 2. Shut the VM down, change it, boot again.

## 5. Importing images

EVE-NG has three separate image subsystems with three separate directories and three sets of rules. Putting a file in the wrong one is the usual reason an image doesn't appear in the node picker.

| Type | Directory | Format |
|---|---|---|
| **QEMU** | `/opt/unetlab/addons/qemu/` | A folder per image, containing `.qcow2` disk(s) |
| **IOL** (IOS on Linux) | `/opt/unetlab/addons/iol/bin/` | Flat `.bin` files, plus an `iourc` license file |
| **Dynamips** | `/opt/unetlab/addons/dynamips/` | Flat classic IOS `.image` / `.bin` files |

Upload with SCP/SFTP — WinSCP or FileZilla to `10.10.20.50` as `root`.

### QEMU images: the naming rules

This is where EVE-NG is strict and unforgiving. Two things must both be right:

**1. The folder name must start with a prefix EVE-NG recognizes**, followed by `-` and then any name/version you like. The prefix is how EVE-NG knows which node template, NIC driver, and boot options to apply. An unrecognized prefix means the image is silently ignored.

```
/opt/unetlab/addons/qemu/<prefix>-<your name and version>/
```

**2. The disk file inside must use one of the supported disk-format names** — the filename tells EVE-NG which virtual disk controller to attach:

| Pattern | Example |
|---|---|
| `hd([a-z]+).qcow2` | `hda.qcow2` |
| `virtio([a-z]+).qcow2` | `virtioa.qcow2` |
| `virtide([a-z]+).qcow2` | `virtidea.qcow2` |
| `scsi([a-z]+).qcow2` | `scsia.qcow2` |
| `sata([a-z]+).qcow2` | `sataa.qcow2` |
| `lsi([a-z]+).qcow2` | `lsia.qcow2` |
| `megasas([a-z]+).qcow2` | `megasasa.qcow2` |

Multi-disk images increment the final letter: `virtioa.qcow2`, `virtiob.qcow2`, `virtioc.qcow2`.

A `.qcow2` downloaded from anywhere will almost never already have the right name. Renaming it is expected, not a workaround.

### Prefixes for the images this lab uses

The full table is on [EVE-NG's Qemu image namings page](https://www.eve-ng.net/index.php/documentation/qemu-image-namings/) and runs to well over a hundred entries. The ones relevant to the labs in [`../Labs/`](../Labs/README.md):

| Prefix | Device | Disk name |
|---|---|---|
| `vios-` | Cisco vIOS L3 router | `virtioa` |
| `viosl2-` | Cisco vIOS L2 switch | `virtioa` |
| `csr1000v-` | Cisco CSR1000v 3.x | `virtioa` |
| `csr1000vng-` | Cisco CSR1000v 16.x / 17.x | `virtioa` |
| `c8000v-` | Catalyst 8000v router | `virtioa` |
| `cat9kv-` | Catalyst 9000v | `virtioa` |
| `nxosv9k-` | Cisco Nexus 9000v | `sataa` (best performance) |
| `xrv-` | Cisco XRv | `hda` |
| `xrv9k-` | Cisco XRv 9000 | `virtioa` |
| `asav-` | Cisco ASAv | `virtioa` |
| `vsrx-` | Juniper vSRX 12.1 | `virtioa` |
| `vsrxng-` | Juniper vSRX 15.x and later | `virtioa` |
| `vmx-` | Juniper vMX router | `hda` |
| `vmxvcp-` | Juniper vMX control plane | `virtioa`, `virtiob`, `virtioc` |
| `vmxvfp-` | Juniper vMX forwarding plane | `virtioa` |
| `vjunosswitch-` | Juniper vJunos switch (vEX) | `virtioa` |
| `vjunosrouter-` | Juniper vJunos router | `virtioa` |
| `vjunosevo-` | Juniper vJunos EVO | `virtioa` |
| `veos-` | Arista vEOS switch | `hda` + `cdrom.iso` |
| `pfsense-` | pfSense firewall | `virtioa` |
| `fortinet-` | Fortinet FortiGate | `virtioa` |
| `paloalto-` | Palo Alto firewall | `virtioa` |
| `linux-` | Any Linux host | `virtioa` |
| `win-` | Windows workstation | `hda`, or `virtioa` with the VirtIO driver |

Worked example — importing a vIOS L2 switch for the VLAN/inter-VLAN routing lab (planned, not yet built):

```
mkdir -p /opt/unetlab/addons/qemu/viosl2-adventerprisek9-15.2
cd /opt/unetlab/addons/qemu/viosl2-adventerprisek9-15.2
mv /root/viosl2-image.qcow2 virtioa.qcow2
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### Always run fixpermissions

```
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

Run it after **every** image addition, no exceptions. Files uploaded over SCP land owned by `root` with whatever mode the client used; EVE-NG's web service runs as a different user and can't read them. An image with correct naming but wrong ownership shows up in the picker and then fails to boot, which is a far more confusing symptom than not showing up at all.

### IOL images

IOL (IOS on Linux, also called IOU) nodes are dramatically lighter than QEMU nodes — tens of megabytes of RAM instead of a gigabyte — which matters when a topology needs eight switches. Rules:

- Files go directly in `/opt/unetlab/addons/iol/bin/`, no per-image folder.
- The filename **must end in `.bin`** and the file must be executable. Newer IOL XE images from Cisco ship with no extension at all; rename them (`x86_64_crb_linux_l2-adventerprisek9-ms` → `x86_64_crb_linux_l2-adventerprisek9-ms.bin`).
- Run `fixpermissions` afterward.
- An `iourc` license file must sit in the same directory. It's bound to the server's hostname *and* domain name, so it breaks if you rename the host later — one more reason to set `eve` / `lab.local` correctly in Section 3 and leave it alone.

Versions EVE-NG documents as known-good:

| Type | Image name |
|---|---|
| L2/L3 switch | `i86bi_linux_l2-adventerprisek9-ms.SSA.high_iron_20190423.bin` |
| L2/L3 switch | `i86bi_LinuxL2-AdvEnterpriseK9-M_152_May_2018.bin` |
| L3 router | `i86bi_LinuxL3-AdvEnterpriseK9-M2_157_3_May_2018.bin` |
| L3 XE router | `x86_64_crb_linux-adventerprisek9-ms.bin` |
| L2/L3 XE switch | `x86_64_crb_linux_l2-adventerprisek9-ms.bin` |

Allocate 1024 MB RAM and 1024 KB NVRAM per IOL node. EVE-NG specifically warns against IOL L3 15.5.2T — it has a console freeze bug after prolonged runtime.

**On licensing:** IOL images and their license files are Cisco-internal software, not licensed for general distribution. EVE-NG's own documentation declines to explain license generation for that reason, and so does this guide. If you don't have a legitimate entitlement, the QEMU route is the honest path — vIOS/IOSvL2 images come with a Cisco Modeling Labs (CML) subscription, and Cisco has made CML-Free and CML-Personal tiers available. Track whichever route you take in [`../Labs/EVE-NG-Images/README.md`](../Labs/EVE-NG-Images/README.md).

### Dynamips images

Classic IOS images for real 7200/3725-class routers go in `/opt/unetlab/addons/dynamips/`. Same `fixpermissions` step. Worth knowing about, but for CCNA-era topics vIOS or IOL is a better fit — Dynamips emulates genuinely old hardware and needs idle-PC tuning to avoid pinning a CPU core.

## 6. When an image doesn't appear

Work down this list in order; it's ordered by how often each one is the cause.

1. **Folder prefix isn't on the supported list.** Check the table above. `ios-` or `cisco-` are not valid prefixes.
2. **Disk file has the wrong name.** `viosl2.qcow2` is wrong; `virtioa.qcow2` is right.
3. **`fixpermissions` wasn't run.** Run it. Then run it again after any change.
4. **Wrong subsystem.** A `.bin` IOL image in the `qemu/` tree, or a `.qcow2` in `iol/bin/`, will never register.
5. **Node added before the image existed.** EVE-NG reads available images when the node is created. Delete the node from the topology and re-add it.
6. **Image is corrupt or truncated.** Verify with `qemu-img info virtioa.qcow2` — it should report a sane virtual size and no errors.

Nodes that appear but won't boot are almost always the nested-virtualization check in Section 4, or a RAM allocation the VM can't satisfy.

## 7. Backups

The EVE-NG VM is worth backing up but is the most expensive thing in the cluster to store, since the disk holds every imported image. Two options, per `BACKUP-SETUP.md`:

- **Full `vzdump` of the VM.** Simplest, and restores in one step. Large — plan retention around the disk size, not the used space.
- **Back up labs and config only, rebuild the rest.** `/opt/unetlab/labs/` holds the topologies (small), and this guide plus [`../Labs/EVE-NG-Images/README.md`](../Labs/EVE-NG-Images/README.md) is enough to rebuild the VM and re-import images from local copies. Cheaper, slower to recover.

Either way the lab topologies themselves should also live in `topology/` in their lab folder under [`../Labs/`](../Labs/README.md), committed to git. That's the real backup — a `.unl` is a few KB of XML and version control beats a nightly dump for something that changes as you work on it.

Node images are excluded from git by `.gitignore` and always will be. They're large and licensed; keep a local copy outside the repo.

## Sources

- [EVE-NG Qemu image namings](https://www.eve-ng.net/index.php/documentation/qemu-image-namings/) — the authoritative prefix and disk-name table
- [EVE-NG: Cisco IOL (IOS on Linux)](https://www.eve-ng.net/index.php/documentation/howtos/howto-add-cisco-iol-ios-on-linux/) — IOL paths, recommended image versions, `.bin` requirement
- [EVE-NG System Requirements](https://www.eve-ng.net/index.php/documentation/installation/system-requirement/) and [Supported Hardware and Software](https://www.eve-ng.net/index.php/supported-hardware-and-software-systems/)
- [EVE-NG Download](https://www.eve-ng.net/index.php/download/) — ISO and the resource calculator
- [Installing EVE-NG Community on Proxmox](https://www.mikelossmann.me/2025/09/26/installing-eve-ng-community-on-proxmox/) — Proxmox VM settings and the multi-reboot install sequence
