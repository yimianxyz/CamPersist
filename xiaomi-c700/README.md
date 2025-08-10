# Xiaomi C700 Camera Persistence Solution

This directory contains documentation for the Xiaomi C700 camera setup installed at parents' home, including details about how the camera recordings are persisted to storage.

## Camera Specifications

- **Model**: Xiaomi C700
- **Location**: Parents' Home
- **Local Storage**: 256GB SD Card (temporary storage)
- **Recording Mode**: Event-based recording (motion detection, human detection, face detection, sound detection, view changes)
- **Daily Data Generation**: ~10GB per day
- **Network Connectivity**: Local WiFi network

## Storage Solution

The storage solution implements a multi-tier architecture:

1. **Tier 1 (Cache)**: 256GB SD Card in camera  
2. **Tier 2 (Local Inbox + Bridge)**: Debian server (`archive.yimian.xyz`) in the same LAN, **with a local “inbox” directory for camera uploads**  
3. **Tier 3 (Long-term Storage)**: Aliyun OSS bucket (`cam-archive`) in Beijing Region (`cn-beijing`) mounted via ossfs

---

### Storage Flow

**Old Flow**: Camera wrote directly to `/mnt/oss` via Samba → this caused OSS GET/list requests during listings.  

**New Flow** (cost-optimized, GET-free for writes):
1. Camera writes new files via SMB to a **local ext4 inbox**: `/srv/camera_inbox`
2. A cron job moves completed files (older than 5 minutes) from the inbox to the OSS mount `/mnt/oss/camera_videos`  
3. OSS lifecycle rules move files to Deep Cold Archive after 30 days

---

### Storage Requirements

- **Daily Storage**: ~10GB/day
- **Monthly Storage**: ~300GB/month
- **Retention Period**: Indefinite
- **Storage Classes**:
  - **Initial**: Standard Storage (¥0.12/GB/month)
  - **Long-term**: Deep Cold Archive (¥0.0075/GB/month)
  - **Cost Optimization**: ~94% reduction after transition

---

### Directory Structure

```

Local Inbox:
/srv/camera\_inbox/
└── XiaomiCamera\_00\_78DF72FFC6DD/

OSS Mount:
/mnt/oss/camera\_videos/
└── XiaomiCamera\_00\_78DF72FFC6DD/

````

---

### Retention Strategy

1. **Permission-Based Protection**:
   - Create a dedicated SMB user for the camera with write-only permissions (no deletes)
   - Even with the 30-day retention setting in the camera, deletion attempts fail

2. **SMB User Setup**:
```bash
sudo useradd -M -s /sbin/nologin xiaomi_cam
sudo smbpasswd -a xiaomi_cam
````

3. **Samba Configuration** (Docker)

   * SMB share now points to **`/srv/camera_inbox`** instead of `/mnt/oss/camera_videos`
   * Write-only semantics (no read access to prevent browsing)

```yaml
  samba:
    container_name: samba
    image: dperson/samba
    hostname: archive-nas
    ports:
      - "139:139/tcp"
      - "445:445/tcp"
      - "137:137/udp"
      - "138:138/udp"
    tmpfs:
      - /tmp
    restart: unless-stopped
    volumes:
      - "/srv/camera_inbox:/camera:rw"
    command: >-
      -u "xiaomi_cam;PASSWORD"
      -s "camera_videos;/camera;yes;no;no;all;none"
      -w "WORKGROUP"
      -n
      -S
```

4. **Local Inbox Permissions**:

```bash
sudo mkdir -p /srv/camera_inbox
sudo chown 1000:1000 /srv/camera_inbox
sudo chmod 773 /srv/camera_inbox
```

---

### Camera-Side Configuration

* In Xiaomi Home APP:

  * Configure NAS storage
  * Set retention to 30 days (deletion prevented by permissions)
  * SMB credentials:

    * Username: `xiaomi_cam`
    * Password: **\[as set above]**
    * Share path: `\\archive.yimian.xyz\camera_videos`

---

## OSS Mount (unchanged from before)

* **Bucket Name**: cam-archive
* **Region**: cn-beijing
* **Mount Point**: `/mnt/oss`
* **Subfolder for videos**: `/mnt/oss/camera_videos`
* **Lifecycle**: Transition to Deep Cold Archive after 30 days

---

## Cron-based Mover Script

**Purpose**:
Avoid any OSS reads by letting the camera write locally, then pushing files to OSS in the background.

**Script Path**: `/home/iotcat/config/cron/camera_mover.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

INBOX="/srv/camera_inbox"
DEST="/mnt/oss/camera_videos"

cd "$INBOX"
find . -type f -mmin +5 -print0 | while IFS= read -r -d '' f; do
  mv -f -- "$f" "$DEST/$f" 2>/dev/null || \
    echo "$(date -Is) failed to move $f (dest folder missing)" >&2
done
```

**Install Script**:

```bash
sudo mkdir -p /home/iotcat/config/cron
sudo tee /home/iotcat/config/cron/camera_mover.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
INBOX="/srv/camera_inbox"
DEST="/mnt/oss/camera_videos"
cd "$INBOX"
find . -type f -mmin +5 -print0 | while IFS= read -r -d '' f; do
  mv -f -- "$f" "$DEST/$f" 2>/dev/null || \
    echo "$(date -Is) failed to move $f (dest folder missing)" >&2
done
EOF
sudo chmod +x /home/iotcat/config/cron/camera_mover.sh
```

**Set up Cron**:

```bash
sudo crontab -e
```

Add:

```
*/5 * * * * /home/iotcat/config/cron/camera_mover.sh
```

---

## Why This Works

* **No OSS GET/list operations from the camera** — it only touches the local inbox.
* **OSS writes only happen from the cron mover**, which never lists or browses the mount, only writes files.
* **Subfolder structure is preserved** (camera’s unique naming ensures no collisions).
* Minimal OSS operations = lower cost.

---

## Monitoring and Maintenance

* Check free space in `/srv/camera_inbox`
* Ensure cron job runs (`grep CRON /var/log/syslog` or `journalctl -u cron`)
* Monitor OSS usage and lifecycle transitions
* Check Samba logs for failed writes

