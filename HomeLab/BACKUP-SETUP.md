# Backup Setup: Proxmox to External NAS

Proxmox backs up over NFS or CIFS to an external NAS, not Proxmox Backup Server. For a 3-node lab this size, a separate PBS instance is more moving parts than the dedup/incremental benefits are worth; plain `vzdump` to NAS-backed storage is simpler and just as reliable with sane retention settings. PBS is still a reasonable upgrade later — it's listed as an option in `DEPLOYMENT-GUIDE.md`'s Additional Nodes table.

## Choosing NFS or CIFS
Either works as a native Proxmox storage type, no OS-level `/etc/fstab` mount needed for either:

| | NFS | CIFS/SMB |
|---|---|---|
| Best fit | Linux-native NAS (TrueNAS, OpenMediaVault, the Pi 5 + SATA HAT box from `SERVICES.md`) | Synology/QNAP appliances, or anything Windows-share-only |
| Auth | IP-based export rules, no credentials stored in Proxmox | Username/password stored in Proxmox's storage config |
| Setup effort | Slightly less: no user account to manage on the NAS side | Slightly more: needs a dedicated backup user, not the NAS admin account |

Default to NFS if the NAS supports it well; fall back to CIFS if it doesn't (some consumer NAS NFS implementations are flaky, SMB tends to be more consistently supported).

## Step 1: Prepare the export on the NAS
1. Create a dedicated dataset/share for Proxmox backups, separate from any media or personal file shares, sized to retention needs (see Step 4).
2. **NFS**: export it with read/write access limited to the 3 Proxmox node IPs (`10.10.20.11`–`.13` per `ADDRESSING.md`), not the whole subnet. Watch for **root squash**: if enabled (often the default), the NAS maps the Proxmox root user's writes down to an unprivileged account and backups fail with permission denied. Either disable root squash for this export or map root to a NAS user with write access to the share.
3. **CIFS**: create a dedicated backup user on the NAS (not the admin account), grant it read/write on the share only, and use that user's credentials in Proxmox, not your personal NAS login.

## Step 2: Add the storage in Proxmox
`Datacenter > Storage > Add`:
- **NFS**: ID (e.g. `nas-backup`), Server = NAS IP, Export = the path from Step 1, Content = **VZDump backup file** only (don't also use it for ISO/disk images, keeps the storage's purpose unambiguous).
- **CIFS/SMB**: ID, Server, Share, Username/Password from Step 1, Content = **VZDump backup file**.

Confirm it shows green/active in `Datacenter > Storage` before continuing.

## Step 3: Create the backup job
`Datacenter > Backup > Add`:
- **Storage**: the target from Step 2.
- **Selection**: all VMs/LXCs worth protecting: EVE-NG VM, the NAS/Jellyfin LXCs, both Pi-hole+Unbound instances (instance A especially, it's the Nebula Sync source of truth), the Home Assistant VM, the Docker host.
- **Schedule**: daily, off-hours (e.g. 02:00), staggered from anything else disk-intensive (EVE-NG lab runs, Jellyfin transcoding).
- **Mode**: Snapshot where supported (LXCs on local-lvm, VMs with the guest agent installed); falls back to Stop mode automatically where snapshot isn't available, which briefly takes the VM/LXC offline during the backup, worth knowing about for anything that needs to stay up.
- **Compression**: zstd (Proxmox's current default, good ratio without being slow).
- **Retention**: something like `keep-daily=7, keep-weekly=4, keep-monthly=3` as a starting point, sized down or up depending on how much space the NAS export can spare (Step 1).

## Step 4: Performance tip for NFS/CIFS targets
If a backup job's `tmpdir` isn't set, `vzdump` may stage its working files on whatever storage is "closest" to the config, which can mean redundant network I/O when the backup target is already the network share. Set **Advanced > tmpdir** on the backup job to a local path (e.g. `/var/tmp`) on the node's own disk; staging locally and only writing the final archive over the network is meaningfully faster on gigabit-class NAS links.

## Step 5: Verify, don't just trust it
1. Run the job manually once (`Datacenter > Backup`, select the job, `Run now`) rather than waiting for the first scheduled run to find out something's wrong.
2. Actually restore one VM or LXC to a test VMID and confirm it boots. A backup you've never test-restored isn't a verified backup — it's an assumption.
3. Spot-check the NAS export's free space after a few cycles to confirm retention is actually pruning old backups, not just accumulating.

## Notes
- Keep NAS credentials (the CIFS backup user's password, and any NFS export details beyond an RFC1918 IP) out of this repo; they belong in a password manager. See the security note in `PFSENSE-SETUP.md`.
- If the external NAS is the Raspberry Pi 5 + Radxa SATA HAT box from `SERVICES.md`'s storage expansion writeup, the same box can serve both media storage and Proxmox backups, just keep the datasets/shares separate so a runaway backup job can't fill the space media needs, or vice versa.
- Revisit Proxmox Backup Server (`DEPLOYMENT-GUIDE.md`'s Additional Nodes table) if the NAS's raw storage use from full `vzdump` archives starts to matter; PBS's dedup/incremental backups use meaningfully less space for the same retention depth.

Sources: [Proxmox VE Storage documentation](https://pve.proxmox.com/pve-docs/chapter-pvesm.html), [Proxmox VE Backup and Restore wiki](https://pve.proxmox.com/wiki/Backup_and_Restore), [How to Back Up Proxmox to NAS](https://blog.gnomeitsolutions.com/proxmox-backup-to-nas/).
