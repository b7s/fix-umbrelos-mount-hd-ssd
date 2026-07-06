# Persistent External Drive Mount on UmbrelOS (survives reboots & OS updates)

UmbrelOS wipes `/etc/systemd/system/` and `/etc/fstab`-style mounts on reboot and on OS updates, so a normal `fstab` entry disappears. This guide uses Umbrel's **official custom pre-start hook** (`/home/umbrel/umbrel/custom-hooks/pre-start`) plus a systemd unit stored in the Umbrel-preserved home directory, so the mount comes back automatically on every boot and after every update. Works for both **NTFS** (Windows drives) and **ext4** (Linux drives).

---

## 1. SSH into the Umbrel box

```bash
ssh umbrel@<UMBREL_IP>
# example: ssh umbrel@192.168.100.37
```

## 2. Identify your disk and its filesystem

```bash
sudo blkid
# or
ls -l /dev/disk/by-uuid/
```

Look for the partition you want to mount. Output will look something like:

```
/dev/sda2: LABEL="midia" UUID="92F6097FF6096537" TYPE="ntfs" PARTLABEL="..."
/dev/sdb1: LABEL="data"  UUID="a1b2c3d4-...."    TYPE="ext4"
```

Note down:
- `UUID` -> use everywhere below
- `TYPE` -> `ntfs` (use NTFS section) or `ext4` (use ext4 section)

## 3. Create the systemd mount unit in the preserved location

`/home/umbrel/umbrel/` is the only path UmbrelOS guarantees to keep across updates, so that's where the unit file lives. Replace `<MOUNT_POINT>`, `<UUID>`, and choose the right block for your filesystem.

- `<MOUNT_POINT>` -> e.g. `/mnt/hd`, `/mnt/data`, `/media/usb`
- `<UUID>`         -> copied from step 2

### Option A: NTFS drive

```bash
MOUNT_POINT=/mnt/hd
UUID=<UUID>

mkdir -p /home/umbrel/umbrel/.systemd
sudo tee /home/umbrel/umbrel/.systemd/force-mount-hd.service >/dev/null <<EOF
[Unit]
Description=Force mount NTFS drive on ${MOUNT_POINT} (survives UmbrelOS updates)
After=local-fs.target
Before=umbrel.service docker.service
Wants=network-online.target
Conflicts=shutdown.target
StartLimitIntervalSec=0

[Service]
Type=forking
GuessMainPID=no
ExecStartPre=/bin/sh -c 'mkdir -p ${MOUNT_POINT} && chown umbrel:umbrel ${MOUNT_POINT} 2>/dev/null; true'
ExecStartPre=/bin/sh -c 'until [ -b /dev/disk/by-uuid/${UUID} ]; do sleep 2; done'
ExecStartPre=/bin/sh -c 'ntfsfix /dev/disk/by-uuid/${UUID} 2>/dev/null; true'
ExecStart=/bin/sh -c 'mountpoint -q ${MOUNT_POINT} || mount -t ntfs-3g -o defaults,nofail,uid=1000,gid=1000,umask=022 /dev/disk/by-uuid/${UUID} ${MOUNT_POINT}'
Restart=on-failure
RestartSec=5s
ExecStop=/bin/sh -c 'umount ${MOUNT_POINT} 2>/dev/null; true'

[Install]
WantedBy=multi-user.target
EOF
```

NTFS specifics:
- `Type=forking` + `GuessMainPID=no` -> systemd (PID 1) owns the mount, so it won't be torn down when a hook exits.
- `ntfsfix` clears the Windows dirty/hibernated flag that silently blocks ntfs-3g.
- `uid=1000,gid=1000` -> `umbrel` user can read/write (default ntfs-3g mounts as root-only).

Make sure the driver is installed:
```bash
sudo apt-get install -y ntfs-3g
```

### Option B: ext4 drive

```bash
MOUNT_POINT=/mnt/hd
UUID=<UUID>

mkdir -p /home/umbrel/umbrel/.systemd
sudo tee /home/umbrel/umbrel/.systemd/force-mount-hd.service >/dev/null <<EOF
[Unit]
Description=Force mount ext4 drive on ${MOUNT_POINT} (survives UmbrelOS updates)
After=local-fs.target
Before=umbrel.service docker.service
Wants=network-online.target
Conflicts=shutdown.target
StartLimitIntervalSec=0

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/sh -c 'mkdir -p ${MOUNT_POINT} && chown umbrel:umbrel ${MOUNT_POINT} 2>/dev/null; true'
ExecStartPre=/bin/sh -c 'until [ -b /dev/disk/by-uuid/${UUID} ]; do sleep 2; done'
ExecStart=/bin/sh -c 'mountpoint -q ${MOUNT_POINT} || mount -t ext4 -o defaults,nofail /dev/disk/by-uuid/${UUID} ${MOUNT_POINT}'
Restart=on-failure
RestartSec=5s
ExecStop=/bin/sh -c 'umount ${MOUNT_POINT} 2>/dev/null; true'

[Install]
WantedBy=multi-user.target
EOF
```

ext4 specifics:
- `Type=oneshot` + `RemainAfterExit=yes` is enough (ext4 is a kernel filesystem, no userspace daemon).
- If you want a specific user/group to own files on the ext4 volume, replace `-o defaults,nofail` with `-o defaults,nofail,uid=1000,gid=1000` (note: this only works if the filesystem was formatted to support it; otherwise set ownership after mounting with `chown -R umbrel:umbrel <MOUNT_POINT>`).

## 4. Install and enable the unit live

```bash
sudo cp /home/umbrel/umbrel/.systemd/force-mount-hd.service /etc/systemd/system/
sudo chown root:root /etc/systemd/system/force-mount-hd.service
sudo chmod 644 /etc/systemd/system/force-mount-hd.service
sudo systemctl daemon-reload
sudo systemctl enable force-mount-hd.service
sudo systemctl restart force-mount-hd.service
sleep 5
mountpoint <MOUNT_POINT> && echo "STILL MOUNTED" || echo "GONE"
findmnt <MOUNT_POINT>
```

If `STILL MOUNTED` is shown, the unit itself works.

## 5. Create the official Umbrel custom pre-start hook

Umbrel's `/opt/umbrel-custom-hooks/run-pre-start` runs `${UMBREL_CUSTOM_PRE_START_HOOK:-/home/umbrel/umbrel/custom-hooks/pre-start}` on every boot, **before umbreld starts**. We put our restore script there so it survives UmbrelOS updates.

```bash
mkdir -p /home/umbrel/umbrel/custom-hooks
sudo tee /home/umbrel/umbrel/custom-hooks/pre-start >/dev/null <<'EOF'
#!/bin/bash
# Restore mount unit if UmbrelOS wiped it, then let systemd do the actual mount.
set -u

UNIT=/etc/systemd/system/force-mount-hd.service
SRC=/home/umbrel/umbrel/.systemd/force-mount-hd.service

if [ ! -f "$UNIT" ] && [ -f "$SRC" ]; then
    cp "$SRC" "$UNIT"
    chmod 644 "$UNIT"
    systemctl daemon-reload
fi

# Make sure the mount point exists
MOUNT_POINT=$(sed -n 's/.*on \([^ ]*\) (survives.*/\1/p' "$SRC" 2>/dev/null | head -1)
[ -z "$MOUNT_POINT" ] && MOUNT_POINT=/mnt/hd
mkdir -p "$MOUNT_POINT"
chown umbrel:umbrel "$MOUNT_POINT" 2>/dev/null || true

# Start (or restart) the mount unit so PID 1 owns the mount, not this hook.
systemctl start force-mount-hd.service 2>/dev/null || true
EOF
sudo chmod +x /home/umbrel/umbrel/custom-hooks/pre-start
sudo chown umbrel:umbrel /home/umbrel/umbrel/custom-hooks/pre-start
```

> If you have **multiple drives**, duplicate the unit with different filenames (e.g. `force-mount-data.service`) and add a `systemctl start` line for each inside `pre-start`. Each unit must point to a unique `<MOUNT_POINT>` and `<UUID>`.

## 6. Add a redundant fstab entry (optional backup layer)

Harmless extra safety; if both systemd and the hook somehow fail, fstab can still pick it up.

For NTFS:
```bash
grep -q '<MOUNT_POINT>' /etc/fstab || echo "UUID=<UUID> <MOUNT_POINT> ntfs-3g defaults,nofail,uid=1000,gid=1000,umask=022 0 0" | sudo tee -a /etc/fstab
```

For ext4:
```bash
grep -q '<MOUNT_POINT>' /etc/fstab || echo "UUID=<UUID> <MOUNT_POINT> ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab
```

## 7. Test by rebooting

```bash
sudo reboot
```

Wait for the box to come back, SSH in again, then:

```bash
mountpoint <MOUNT_POINT> && echo "STILL MOUNTED" || echo "GONE"
systemctl status force-mount-hd.service --no-pager
journalctl -u force-mount-hd.service -b --no-pager | tail -20
journalctl -u umbrel-custom-pre-start.service -b --no-pager | tail -20
```

For NTFS you should see messages like `Mounted /dev/sdXN (Read-Write, ...)` (no `Unmounting` right after). For ext4 you'll just see the unit become `active (exited)`. In both cases `mountpoint <MOUNT_POINT>` should report `is a mountpoint`.

---

## How the three layers work together

| Layer | What it does | Why it exists |
|---|---|---|
| `/etc/fstab` entry | Mount at boot, fallback | Standard Linux; harmless backup |
| `/etc/systemd/system/force-mount-hd.service` (PID 1 owned) | Actually performs the mount and supervises it | For NTFS: `Type=forking` lets systemd own ntfs-3g so it persists. For ext4: `Type=oneshot` is enough. Runs before umbrel/docker. |
| `/home/umbrel/umbrel/custom-hooks/pre-start` | Restores the unit if UmbrelOS wiped it, then triggers systemd | UmbrelOS preserves `/home/umbrel/umbrel/` across updates, so the hook and the unit source file survive every OS upgrade |

## Common troubleshooting

**Mount is GONE after reboot and `force-mount-hd.service` is inactive:**
```bash
sudo systemctl restart umbrel-custom-pre-start.service
sleep 5
mountpoint <MOUNT_POINT>
journalctl -u umbrel-custom-pre-start.service -b --no-pager | tail
```
The hook log should say `running '/home/umbrel/umbrel/custom-hooks/pre-start'` and `completed successfully`.

**NTFS: `Mounted /dev/sdXN ... Unmounting /dev/sdXN` in the pre-start log (mount vanishes right after the hook finishes):**
That happens if the mount was done **inside** the hook script (ntfs-3g is torn down when the hook process exits). Make sure your hook only calls `systemctl start force-mount-hd.service`, never `mount` directly. The hook in this guide already does that correctly.

**Wrong UUID / disk not found:**
```bash
sudo blkid
ls -l /dev/disk/by-uuid/
```
Replace `<UUID>` everywhere (in the unit file, the hook, and fstab) with the real UUID.

**NTFS: `ntfs-3g` refuses to mount ("dirty" / "Windows hibernated" / "metadata kept in Windows cache"):**
The unit already runs `ntfsfix` in `ExecStartPre`. If you still see this, on a Windows PC safely eject the drive, or fully shut down Windows (disable Fast Startup) before reconnecting the drive to Umbrel.

**NTFS: `ntfs-3g` not installed:**
```bash
sudo apt-get install -y ntfs-3g
```

**ext4: `mount: wrong fs type, bad option, bad superblock`:**
```bash
sudo fsck /dev/disk/by-uuid/<UUID>
```
Then `sudo systemctl restart force-mount-hd.service`.

**ext4: files are owned by root:**
ext4 stores ownership on the filesystem itself. After mounting, set ownership:
```bash
sudo chown -R umbrel:umbrel <MOUNT_POINT>
```

## Files used

- `/home/umbrel/umbrel/.systemd/force-mount-hd.service` (source, preserved)
- `/etc/systemd/system/force-mount-hd.service` (live copy, auto-restored by the hook)
- `/home/umbrel/umbrel/custom-hooks/pre-start` (Umbrel official pre-start hook, preserved)
- `/etc/fstab` (redundant backup entry)

## To undo

```bash
sudo systemctl disable --now force-mount-hd.service
sudo rm -f /etc/systemd/system/force-mount-hd.service
sudo systemctl daemon-reload
sudo umount <MOUNT_POINT> 2>/dev/null
rm -f /home/umbrel/umbrel/custom-hooks/pre-start
rm -f /home/umbrel/umbrel/.systemd/force-mount-hd.service
# Remove the /etc/fstab line:
sudo sed -i '\#<MOUNT_POINT>#d' /etc/fstab
```
