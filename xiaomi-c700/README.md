
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
2. **Tier 2 (Bridge)**: Debian server (archive.yimian.xyz) in the same LAN
3. **Tier 3 (Long-term Storage)**: Aliyun OSS bucket (cam-archive) in Beijing Region (cn-beijing)

### **Updated Storage Flow for Cost Optimization**
> **Change from original design**: Instead of writing directly to the OSS mount via Samba (which triggered GET/list requests), the camera now writes to a **local inbox directory** on the Debian server. A cron job moves completed files from the inbox to the OSS mount without listing or reading from OSS.

**New Storage Flow:**
1. Camera records events to SD card
2. Camera connects to local Debian server via SMB v1
3. Samba share points to `/srv/camera_inbox` (local ext4)
4. Cron job moves completed files (older than 5 minutes) from `/srv/camera_inbox` to `/mnt/oss/camera_videos`
5. Debian server mounts Aliyun OSS bucket using ossfs
6. Videos are stored in Standard storage class (LRS)
7. After defined period, automatically transitioned to Deep Cold Archive via lifecycle rules

---

### Storage Requirements

- **Daily Storage**: ~10GB per day
- **Monthly Storage**: ~300GB per month
- **Retention Period**: Indefinite (permanent storage)
- **Storage Classes**:
  - **Initial Storage**: Standard Storage (¥0.12/GB/month)
  - **Long-term Storage**: Deep Cold Archive (¥0.0075/GB/month)
  - **Cost Optimization**: ~94% cost reduction after transition to Deep Cold Archive

### Directory Structure
```

Local Inbox:
/srv/camera\_inbox/
└── XiaomiCamera\_00\_78DF72FFC6DD/

OSS Mount:
/mnt/oss/
└── camera\_videos/
└── XiaomiCamera\_00\_78DF72FFC6DD/

````

---

### Retention Strategy
1. **Permission-Based Protection**:
   - Create a dedicated SMB user for the camera
   - Configure write-only permissions (no delete permissions)
   - Camera can write new files but cannot delete existing ones
   - Even with 30-day retention setting in camera, deletion attempts will fail

2. **SMB User Setup**:
```bash
# Create dedicated user for camera
sudo useradd -M -s /sbin/nologin xiaomi_cam
sudo smbpasswd -a xiaomi_cam
````

3. **Samba Configuration** (in Docker, updated to use local inbox):

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
      - "/srv/camera_inbox:/camera:rw"   # <-- now points to local inbox
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

### Camera-Side Configuration:

* Configure NAS storage in Xiaomi Home APP
* Set retention to 30 days (deletion will be prevented by permissions)
* Use the following credentials:

  * Username: xiaomi\_cam
  * Password: \[as set by smbpasswd]
  * Share path: \archive.yimian.xyz\camera\_videos

---

### Storage Cost Estimation

* **First Month (Standard Storage)**: 300GB × ¥0.12 = ¥36/month
* **Long-term Monthly (Deep Cold Archive)**: 300GB × ¥0.0075 = ¥2.25/month
* **Yearly Accumulation**: \~3.6TB
* **Example Annual Cost After 1 Year**:

  * Recent month in Standard: 300GB × ¥0.12 = ¥36
  * Previous 11 months in Deep Cold Archive: 3.3TB × ¥0.0075 = ¥24.75
  * Total: \~¥60.75/month

---

### OSS Bucket Configuration

* **Bucket Name**: cam-archive
* **Region**: China North 2 (Beijing)
* **Storage Class**: Standard Storage
* **Redundancy Type**: Locally Redundant Storage (LRS)
* **Access Control**: Private
* **Endpoint**:

  * Internal: `oss-cn-beijing-internal.aliyuncs.com`
  * External: `oss-cn-beijing.aliyuncs.com`

---

## Implementation

### Prerequisites

1. Xiaomi C700 camera with 256GB SD card
2. Debian server (archive.yimian.xyz) in the same LAN
3. Aliyun OSS bucket
4. Aliyun OSS access credentials
5. FUSE 2.8.4+ installed on Debian server

### Setup Steps

#### 1. Aliyun OSS Setup

* Create OSS bucket
* Generate access credentials
* Install ossfs on Debian server
* Configure ossfs mount point
* Configure lifecycle rules for transition to Deep Cold Archive

#### 2. Debian Server Setup

* Install Samba server
* Create xiaomi\_cam user
* Configure SMB share with write-only permissions pointing to `/srv/camera_inbox`
* Mount OSS bucket to `/mnt/oss` using ossfs
* Set appropriate directory permissions

#### 3. Camera Setup

* Configure NAS storage in Xiaomi Home APP
* Use provided xiaomi\_cam credentials
* Test write access

---

### OSS Lifecycle Rules

```
Rule 1: Transition to Deep Cold Archive
- Prefix: videos/
- Days after object creation: 30
- Transition to: Deep Cold Archive Storage
```

---

### ossfs Configuration

> Reference:
>
> * [Installing ossfs](https://help.aliyun.com/zh/oss/developer-reference/installing-ossfs)
> * [Advanced Configuration](https://help.aliyun.com/zh/oss/developer-reference/advanced-configurations)

**Storage Behavior**:

* Files in `/mnt/oss` are virtual representations of OSS objects
* No significant local disk space is used
* Only metadata and transfer buffers are stored locally
* Actual video data is stored in Aliyun OSS
* Server acts as a pass-through between camera and OSS

*(Full ossfs install and mount commands remain as in original README)*

---

## Cron-based Mover Script

**Purpose**: Avoid OSS GET/list during camera uploads by staging to local inbox first.

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

## Monitoring and Maintenance

* Monitor available space on SD card
* Monitor `/srv/camera_inbox` space to ensure mover keeps up
* Monitor ossfs mount status
* Monitor Aliyun OSS usage and costs
* Verify lifecycle rule execution
* Verify SMB permissions are maintained
* Monitor failed deletion attempts in Samba logs

---

## Troubleshooting

Common issues:

* ossfs connection drops
* SMB connection issues
* Storage capacity issues
* Network bandwidth constraints
* Storage class transition delays
* Deep Cold Archive retrieval times (12/48 hours)

---

## Future Improvements

* Implement monitoring and alerting system
* Optimize storage transition periods
* Consider implementing backup solution
* Add data retrieval automation tools
* Implement video indexing and search capabilities


