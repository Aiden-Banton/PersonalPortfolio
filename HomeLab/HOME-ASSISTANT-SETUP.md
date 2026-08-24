# Home Assistant Setup

Home Assistant OS (HAOS) running in a Proxmox VM on Node 3, with add-ons enabled through HAOS's built-in Supervisor.

## Why HAOS-in-a-VM, not "Supervised"
I'd originally planned to use the Supervised install method (your own Debian with the Supervisor on top). Home Assistant deprecated that method starting with the 2025.6 release, and it's been unsupported since 2025.12. Their recommended replacement is **Home Assistant OS**, which the project states "supports everything Supervised does, including add-ons," run either on dedicated hardware or in a VM. Since this lab is already virtualized, HAOS in a Proxmox VM is the direct swap: same add-on ecosystem, same functionality, on a currently-maintained install method. The Container install method (Core only, no Supervisor) was ruled out because it drops the add-on store, which was the whole point of wanting Supervised in the first place.

## Before you start
- Node 3 has capacity per the budget in `SERVICES.md` (target: 4GB RAM, 2 vCPU for the VM).
- VLAN 20 is normally static-only (`ADDRESSING.md`); this guide temporarily enables DHCP on it for first boot only, then reverts that.
- `10.10.20.30` is reserved for this VM per `ADDRESSING.md`.

## Step 1: Download the HAOS image
On a machine with internet access (or directly on the Proxmox node via shell):
1. Go to https://github.com/home-assistant/operating-system/releases/latest and grab the file ending in `haos_ova-<version>.qcow2.xz` — this is the generic KVM/QEMU image, the one that works in Proxmox.
2. Copy it to the Proxmox node (e.g. `scp` to `/var/lib/vz/template/iso/` or straight into the Proxmox shell) and decompress it:
   ```
   xz -d haos_ova-*.qcow2.xz
   ```

## Step 2: Create the VM shell
In the Proxmox web UI, `Create VM`:
- **General**: VM ID per your numbering, Name `home-assistant`.
- **OS**: "Do not use any media."
- **System**: BIOS = **OVMF (UEFI)**, add an **EFI disk**, uncheck "Pre-Enroll keys" (HAOS isn't signed for Secure Boot). Machine = **q35**.
- **Disks**: delete/skip the default disk the wizard adds; the HAOS image replaces it in Step 3.
- **CPU**: 2 vCPU, type `host`.
- **Memory**: 4096 MB.
- **Network**: bridge `vmbr0`, VLAN tag `20`, model VirtIO.

Don't start the VM yet.

## Step 3: Import the HAOS disk
From the Proxmox node's shell:
```
qm importdisk <vmid> haos_ova-*.qcow2 local-lvm
```
Then in the VM's **Hardware** tab:
1. The import creates an "Unused Disk." Double-click it, set **Bus/Device** to **SATA0** (HAOS requires SATA, VirtIO/SCSI won't boot), confirm.
2. **Options > Boot Order**: enable `sata0`, disable everything else.
3. Start the VM.

## Step 4: First boot and network handoff
HAOS looks for DHCP on first boot, but VLAN 20 is static-only by design (`ADDRESSING.md`). Rather than break that policy permanently:
1. Temporarily enable a DHCP pool on the PVE tab (`Services > DHCP Server` in pfSense; see `ADDRESSING.md`'s note that this VLAN "can add a pool later for flexibility").
2. Boot the VM, watch the console until it reports a reachable IP, then browse to `http://<that-ip>:8123` and complete onboarding (account, location, etc.).
3. In the HA web UI: `Settings > System > Network`, switch from DHCP to static, set `10.10.20.30/24`, gateway `10.10.20.1`. DNS can stay blank/default for now; once `DNS-HA-SETUP.md` is deployed, add `10.10.20.20` there too.
4. Disable the temporary DHCP pool on the PVE VLAN again, back to static-only.

## Step 5: (Optional) USB passthrough for Zigbee/Z-Wave
If a Zigbee or Z-Wave USB dongle is in play: Proxmox web UI, VM's **Hardware > Add > USB Device**, choose **Use USB Vendor/Device ID** (not "Port," which changes if the stick moves to a different physical port), select the dongle. HA will see it as a normal USB device inside the VM once it's next rebooted.

## Recommended add-ons
Installed from **Settings > Add-ons > Add-on Store** inside HA, no separate download needed:

| Add-on | Why |
|---|---|
| Terminal & SSH | Basic in-VM administration without needing Proxmox console access every time |
| File editor | Direct YAML edits (automations, `configuration.yaml`) from the browser |
| Mosquitto broker | MQTT broker, prerequisite for most non-Zigbee2MQTT integrations that speak MQTT |
| Node-RED | Visual automation editor, worth it once automations get more complex than the built-in UI handles well |
| ESPHome | Only relevant if flashing/managing ESP32-based sensors; skip if not using them |
| Zigbee2MQTT | Only relevant with a Zigbee USB dongle (Step 5); alternative to HA's built-in ZHA integration if broader device support is needed |

**Pi-hole integration** (not an add-on, a built-in integration): `Settings > Devices & Services > Add Integration > Pi-hole`, point it at the DNS VIP (`10.10.20.20`) once `DNS-HA-SETUP.md` is live. Surfaces block counts and lets HA disable Pi-hole temporarily from a dashboard, small payoff for something already running on the network.

## Notes
- Home Assistant lives on **Node 3**, not the M910q. EVE-NG stays the only thing on the M910q; see the "why" note in `SERVICES.md`.
- Keep the HA `secrets.yaml` (API keys, integration credentials) out of this repo, even if the rest of the config is worth version-controlling privately. See the security note in `PFSENSE-SETUP.md`.
- Update this file's "Status" note (once added to `SERVICES.md`'s table) to `running` after Step 4 completes successfully.

Sources: [Deprecating Core and Supervised installation methods, and 32-bit systems](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/), [Home Assistant: Alternative installation methods](https://www.home-assistant.io/installation/alternative/), [Proxmox forum: Install Home Assistant OS in a VM](https://forum.proxmox.com/threads/guide-install-home-assistant-os-in-a-vm.143251/).
