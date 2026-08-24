# Hardware Acceleration: Jellyfin Transcoding

Passing the M700's integrated GPU into the Jellyfin LXC (Node 2) for Intel Quick Sync (QSV) transcoding, so Jellyfin isn't burning the i3-6100T's 2 cores on software transcoding.

## What the hardware actually supports
The i3-6100T's HD Graphics 530 (Skylake, 6th gen) supports Quick Sync for:

| Codec | Decode | Encode |
|---|---|---|
| H.264 | Yes | Yes |
| HEVC (H.265), 8-bit | Yes | Yes |
| HEVC (H.265), 10-bit | No (needs 7th gen/Kaby Lake or newer) | No |
| VP9 | No | No |
| AV1 | No | No |

Practical implication: 4K HDR content is almost always 10-bit HEVC, so anything that triggers a transcode on that content (not direct play, transcodes specifically) falls back to slow software transcoding regardless of this setup. This matters less than it sounds: most clients that support HEVC at all can direct-play it without transcoding. QSV only comes into play when a transcode is actually triggered (bitrate too high for the network, client doesn't support the source codec, etc.).

## Step 1: Confirm the iGPU is visible on the Proxmox host
On Node 2's shell:
```
ls /dev/dri
```
Should list `card0` and `renderD128`. This comes from the `i915` kernel module, which Proxmox loads automatically for Intel iGPUs, no extra host-side driver install needed for the passthrough itself.

Note the group IDs, needed in Step 2:
```
getent group video    # commonly 44
getent group render   # commonly 104
```

## Step 2: Pass the device into the LXC
**If Jellyfin was deployed via the Community Scripts one-liner** (`DEPLOYMENT-GUIDE.md` Step 6.3), it prompts for GPU passthrough interactively during install and configures this automatically; skip to Step 3 and just verify it worked.

**Manual configuration**, if it wasn't set up during install or needs redoing:
1. Edit the container's config on the Proxmox host: `/etc/pve/lxc/<vmid>.conf`, add:
   ```
   dev0: /dev/dri/card0,gid=44
   dev1: /dev/dri/renderD128,gid=104
   ```
   Use the actual GIDs from Step 1 if they differ from the common defaults. Proxmox handles the unprivileged-container UID/GID remapping automatically with this `devN:` syntax; no manual `lxc.cgroup2` or bind-mount entries needed on current Proxmox versions.
2. Inside the container, confirm the `video`/`render` groups exist with matching GIDs (`getent group video render`), then add the Jellyfin service user to both:
   ```
   usermod -aG video,render jellyfin
   ```
3. Restart the container: `pct reboot <vmid>`.
4. Confirm from inside the container: `ls -la /dev/dri` should show both devices, owned by the groups from Step 1.

## Step 3: Enable it in Jellyfin
`Dashboard > Playback > Transcoding`:
- **Hardware acceleration**: Intel QuickSync (QSV)
- **VA-API device**: `/dev/dri/renderD128`
- Enable hardware decoding for H.264 and HEVC (leave VP9/AV1 unchecked, not supported per the table above).
- Enable hardware encoding.
- Leave tone-mapping off initially. Gen9 iGPUs like the HD 530 don't reliably support Jellyfin's OpenCL/VPP-based tone-mapping path; if HDR-to-SDR tone-mapped transcodes matter, test it deliberately and expect it may silently fall back to software or fail rather than assume it works.

## Step 4: Verify it's actually being used
1. Install `vainfo` inside the container (`apt install vainfo`) and run it; it should list supported H.264/HEVC profiles without errors, confirming the device and permissions are correct end to end.
2. Start a playback session that forces a transcode (a client/bitrate combination Jellyfin won't direct-play), then check Jellyfin's **Dashboard > Activity** or the active sessions view, it should tag the session "Transcode (hw)" rather than plain "Transcode."
3. On the Proxmox host, `intel_gpu_top` during that session should show non-zero engine usage, confirming the iGPU is doing the work, not the CPU cores.

## Troubleshooting
- If Jellyfin falls back to software silently: check its transcoding log for the `ffmpeg` command it ran, it'll show whether `-hwaccel vaapi` was actually applied.
- Permission errors on `/dev/dri/*` inside the container almost always mean the GID in the LXC config doesn't match the GID inside the container, recheck Step 2.
- If the LXC is privileged rather than unprivileged, the `devN:` syntax still works the same way; privileged containers just skip the UID/GID remapping Proxmox does for unprivileged ones.

Sources: [Hardware transcoding in Jellyfin on Proxmox with Intel Integrated Graphics](https://erikdevries.com/posts/hardware-transcoding-in-jellyfin-on-proxmox-with-intel-integrated-graphics), [Jellyfin QuickSync in a Proxmox Unprivileged LXC](https://diymediaserver.com/post/jellyfin_intel_quicksync_unprivileged_lxc/), [Proxmox forum: LXC iGPU passthrough guide](https://forum.proxmox.com/threads/proxmox-lxc-igpu-passthrough.141381/).
