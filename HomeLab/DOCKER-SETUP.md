# Docker Host Setup

A general-purpose Docker host on Node 2 (`10.10.20.40`) for self-hosted apps that don't warrant their own dedicated VM. Default is an **unprivileged LXC**; the fallback is a **small VM** if a workload doesn't play well with that.

## LXC vs VM: the actual tradeoff
- Proxmox doesn't officially test or support Docker running inside an LXC. Most "production" guidance says to run Docker in a VM, full stop.
- In practice, Docker-in-LXC is extremely common in homelabs specifically because of the resource gap: an LXC's baseline overhead is roughly 100–500MB RAM, a VM's is 2–4GB, before either one runs a single container. On a 2-core/4-thread i3-6100T that's already carrying Jellyfin, the NAS share, and half the DNS HA pair (`SERVICES.md`'s Node 2 budget), that gap is the difference between comfortable and tight.
- The known LXC-specific failure modes (storage driver issues, UID/GID mapping conflicts in unprivileged containers) are solvable, just extra steps most VM deployments skip entirely.

**Default here: unprivileged LXC.** If a specific workload turns out to need something LXC nesting can't provide, or troubleshooting a storage/permission issue stops being worth the time, that one workload moves to the VM path — not the whole Docker host.

## Decision criteria (when to escalate from LXC to VM)
Start with the LXC. Move to a VM if any of these show up:
- A container image needs a kernel module or feature the LXC's shared host kernel doesn't expose (rare for typical self-hosted apps, more common for things like VPN/firewall containers or anything doing raw network manipulation).
- Node 2's actual CPU/RAM usage (Proxmox's per-node graphs, or `top` inside the LXC) shows consistent contention once real workloads are running, not just the LXC's baseline footprint.
- A workload needs a hard security boundary (e.g. running something less trusted), where LXC's shared-kernel isolation isn't enough.
- Storage driver problems (see Step 3) don't resolve cleanly after the `fuse-overlayfs` fallback.

## Step 1: Create the LXC
- Debian 12, unprivileged, 2 vCPU, 2GB RAM to start (per-container needs are additive on top, see `SERVICES.md`), 16GB+ disk depending on what's deployed.
- Static IP `10.10.20.40/24`, gateway `10.10.20.1`.
- **Enable Nesting and Keyctl** before first boot: `Resources` won't show this, it's under the container's **Options > Features**, or via CLI:
  ```
  pct set <vmid> --features nesting=1,keyctl=1
  ```
  Docker won't run inside the LXC without both.

## Step 2: Install Docker
Standard install, no LXC-specific steps here:
```
curl -fsSL https://get.docker.com | sh
```
or follow the apt-repository method in Docker's own docs if the convenience script isn't preferred: https://docs.docker.com/engine/install/debian/

Test: `docker run hello-world`. If it works, skip Step 3.

## Step 3: If the storage driver fails
Unprivileged nested LXCs sometimes can't use Docker's default `overlay2` storage driver (overlay-on-overlay isn't always supported by the underlying kernel/filesystem combination). Symptom: `docker run` fails with storage-driver errors even though the daemon starts.

Fix: switch to `fuse-overlayfs`.
```
apt install fuse-overlayfs
```
`/etc/docker/daemon.json`:
```json
{
  "storage-driver": "fuse-overlayfs"
}
```
`systemctl restart docker`, retest `docker run hello-world`.

## Step 4: UID/GID mapping for bind mounts
Unprivileged containers remap the root UID/GID range, which can surface as permission-denied errors on bind-mounted volumes even when the container's internal `docker run` command looks correct. If a container can't write to a mounted host path, `chown` that path (on the Proxmox host, in the LXC's actual filesystem) to match the UID the containerized process runs as, rather than assuming it's a Docker Compose config problem.

## VM fallback (if a workload needs it)
No LXC-specific quirks here, it's a normal Debian VM:
1. Create a Debian 12 VM, 2 vCPU, 2–4GB RAM, static IP on VLAN 20.
2. Install Docker the same way as Step 2.
3. Move only the workload that needed escalating; leave everything else on the LXC rather than migrating the whole Docker host over pre-emptively.

## Notes
- Firewall exposure for whatever ends up running here follows the same pattern as EVE-NG and the other exposed services: add a dedicated alias in pfSense per app, not a blanket rule for the whole Docker host. See `ADDRESSING.md` and `PFSENSE-SETUP.md` Step 7 for the pattern.
- Keep any `docker-compose.yml` or `.env` file that carries credentials, API keys, or tokens out of this repo; a sanitized compose file (no secrets) is fine to document. See the security note in `PFSENSE-SETUP.md`.
- Update `SERVICES.md`'s status column once this is actually running, and note here if a workload gets escalated to the VM path, that's worth a one-line record of why.

Sources: [Docker in Proxmox LXC: Nesting & Keyctl Required?](https://proxmox.rdem-systems.com/en/blog/docker-vm-vs-lxc-proxmox/), [Docker on Proxmox VE: VM or LXC Container Guide](https://www.it-connect.tech/how-to-run-docker-containers-on-proxmox-ve/), [Docker Engine install docs (Debian)](https://docs.docker.com/engine/install/debian/).
