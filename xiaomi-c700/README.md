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

### Storage Flow
1. Camera records events to SD card
2. Camera connects to local Debian server via SMB v1
3. Debian server mounts Aliyun OSS bucket using ossfs
4. Videos are stored in Standard storage class (Local Redundancy Storage, LRS)
5. After defined period, automatically transitioned to Deep Cold Archive storage class via lifecycle rules

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
/mnt/oss/
└── camera_videos/      # Directory exposed to camera via SMB (write-only for camera)
```

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
```

3. **SMB Configuration** (in `/etc/samba/smb.conf`):
```ini
[camera_videos]
    path = /mnt/oss/camera_videos
    valid users = xiaomi_cam
    read only = no
    create mask = 0644
    directory mask = 0755
    delete readonly = no
    force user = xiaomi_cam
```

4. **Directory Permissions**:
```bash
# Set directory ownership and permissions
sudo chown root:xiaomi_cam /mnt/oss/camera_videos
sudo chmod 773 /mnt/oss/camera_videos
```

### Camera-Side Configuration:
- Configure NAS storage in Xiaomi Home APP
- Set retention to 30 days (deletion will be prevented by permissions)
- Use the following credentials:
  - Username: xiaomi_cam
  - Password: [as set by smbpasswd]
  - Share path: \\archive.yimian.xyz\camera_videos

### Storage Cost Estimation
- **First Month (Standard Storage)**: 300GB × ¥0.12 = ¥36/month
- **Long-term Monthly (Deep Cold Archive)**: 300GB × ¥0.0075 = ¥2.25/month
- **Yearly Accumulation**: ~3.6TB
- **Example Annual Cost After 1 Year**:
  - Recent month in Standard: 300GB × ¥0.12 = ¥36
  - Previous 11 months in Deep Cold Archive: 3.3TB × ¥0.0075 = ¥24.75
  - Total: ~¥60.75/month

### OSS Bucket Configuration
- **Bucket Name**: cam-archive
- **Region**: China North 2 (Beijing)
- **Storage Class**: Standard Storage
- **Redundancy Type**: Locally Redundant Storage (LRS)
- **Access Control**: Private
- **Endpoint**: 
  - Internal: `oss-cn-beijing-internal.aliyuncs.com`
  - External: `oss-cn-beijing.aliyuncs.com`

## Implementation

### Prerequisites

1. Xiaomi C700 camera with 256GB SD card
2. Debian server (archive.yimian.xyz) in the same LAN
3. Aliyun OSS bucket
4. Aliyun OSS access credentials
5. FUSE 2.8.4+ installed on Debian server

### Setup Steps

1. **Aliyun OSS Setup**
   - Create OSS bucket
   - Generate access credentials
   - Install ossfs on Debian server
   - Configure ossfs mount point
   - Configure lifecycle rules for transition to Deep Cold Archive

2. **Debian Server Setup**
   - Install Samba server
   - Create xiaomi_cam user
   - Configure SMB share with write-only permissions
   - Mount OSS bucket to shared directory using ossfs
   - Set appropriate directory permissions

3. **Camera Setup**
   - Configure NAS storage in Xiaomi Home APP
   - Use provided xiaomi_cam credentials
   - Test write access

### Configuration

#### Required Credentials
Before proceeding with ossfs installation, gather the following information:
1. **OSS Access Credentials**:
   - Access Key ID from Aliyun console
   - Access Key Secret from Aliyun console
2. **Bucket Information** (already configured):
   - Bucket Name: `cam-archive`
   - Region: `cn-beijing`
   - Internal Endpoint: `oss-cn-beijing-internal.aliyuncs.com`

#### OSS Lifecycle Rules
```
Rule 1: Transition to Deep Cold Archive
- Prefix: videos/
- Days after object creation: 30
- Transition to: Deep Cold Archive Storage
```

#### ossfs Configuration
> Reference: 
> - [Installing ossfs](https://help.aliyun.com/zh/oss/developer-reference/installing-ossfs?spm=a2c4g.11186623.0.0.5e923c93kOrtmU)
> - [Advanced Configuration](https://help.aliyun.com/zh/oss/developer-reference/advanced-configurations?spm=a2c4g.11186623.0.0.8cfd81c1FPSxcd)

**Storage Behavior**:
- Files in `/mnt/oss` are virtual representations of OSS objects
- No significant local disk space is used on the CentOS server
- Only metadata and temporary transfer buffers are stored locally
- Actual video data is stored directly in Aliyun OSS
- Server acts as a pass-through for data between camera and OSS

```bash
# 1. Install ossfs
## First install FUSE
sudo yum install fuse fuse-devel

## Check FUSE version (requirement: >= 2.8.4)
fusermount -V

## Install ossfs for CentOS 7
sudo wget https://gosspublic.alicdn.com/ossfs/ossfs_1.91.4_centos7.0_x86_64.rpm
sudo yum install ossfs_1.91.4_centos7.0_x86_64.rpm

## Install mime support (required for proper Content-Type setting)
sudo yum install mailcap

# 2. Verify installation
ossfs --version

# 3. Configure credentials
## Save account information to /etc/passwd-ossfs and set permissions to 640
echo "cam-archive:$ACCESS_KEY_ID:$ACCESS_KEY_SECRET"  | sudo tee /etc/passwd-ossfs

# 4. Create mount point
sudo mkdir -p /mnt/oss

# 5. Mount OSS bucket
## Using debug mode for initial testing
sudo ln -s /usr/local/bin/ossfs /usr/bin/ossfs
sudo ossfs cam-archive /mnt/oss \
    -ourl=https://oss-cn-beijing.aliyuncs.com \
    -oallow_other

# 6. Once testing is successful, add to /etc/fstab for auto-mount on boot (CentOS 7.0+)

## First create the auto-mount script
sudo vim /etc/init.d/ossfs
# Add the following content (modify paths as needed):
########################### File content ###########################

#! /bin/bash
#
# ossfs      Automount Aliyun OSS Bucket in the specified direcotry.
#
# chkconfig: 2345 90 10
# description: Activates/Deactivates ossfs configured to start at boot time.

ossfs cam-archive /mnt/oss -ourl=https://oss-cn-beijing.aliyuncs.com -oallow_other


########################## End of file content ###########################

## Make the script executable
sudo chmod a+x /etc/init.d/ossfs

## Enable the service
sudo chkconfig ossfs on

# 7. Verify mount
df -h
ls /mnt/oss
# 8. To unmount if needed
sudo umount /mnt/oss
```

#### Samba Configuration (Docker)
> Note: Using existing dperson/samba Docker container in docker-compose.yml

1. **First mount OSS on host**:
```bash
# Follow the ossfs configuration steps above to mount OSS at /mnt/oss
```

2. **Update docker-compose.yml Samba service**:
```yaml
  samba:
    container_name: samba
    image: dperson/samba
    hostname: archive-nas
    ports:
      - "139:139/tcp"
      - "445:445/tcp"
      - "137:137/udp"   # Add NetBIOS name service
      - "138:138/udp"   # Add NetBIOS datagram service
    tmpfs:
      - /tmp
    restart: unless-stopped
    volumes:
      - "/home/iotcat/data/nextcloud/data/xiuju/files/家庭共享:/mount"
      - "/mnt/oss/camera_videos:/camera:rw"
    command: >-
      -u "iotcat;123" 
      -s "camera_videos;/camera;yes;no;no;all;none" 
      -w "WORKGROUP" 
      -n
      -S
```

3. **Directory Permissions on Host**:
```bash
# Set permissions on host directory
sudo mkdir -p /mnt/oss/camera_videos
sudo chown 1000:1000 /mnt/oss/camera_videos
sudo chmod 773 /mnt/oss/camera_videos
```

4. **Apply Changes**:
```bash
# Restart the Samba container
docker-compose restart samba

# Test connection
smbclient //archive.yimian.xyz/camera_videos -U iotcat
```

**Additional Network Configuration**:
1. **DNS Entry** (if you have control over local DNS):
```bash
# Add to your DNS server or local hosts file
archive.yimian.xyz  192.168.x.x  # Replace with actual IP
ARCHIVENAS         192.168.x.x  # Replace with actual IP
```

2. **Network Discovery Tips**:
- The camera should now be able to find the server as "ARCHIVENAS" in the network
- Try both server names when configuring the camera:
  1. ARCHIVENAS
  2. archive.yimian.xyz
  3. \\ARCHIVENAS\camera_videos
  4. \\archive.yimian.xyz\camera_videos

3. **Troubleshooting Network Discovery**:
```bash
# Check if NetBIOS name is broadcast correctly
nmblookup ARCHIVENAS

# Check network visibility
smbclient -L ARCHIVENAS -N

# Test connection with NetBIOS name
smbclient //ARCHIVENAS/camera_videos -U iotcat
```

### Camera-Side Configuration:
- Configure NAS storage in Xiaomi Home APP
- Set retention to 30 days (deletion will be prevented by permissions)
- Use the following credentials:
  - Username: xiaomi_cam
  - Password: [as set in docker-compose]
  - Share path: \\archive.yimian.xyz\camera_videos

**Note**: The changes above:
1. Add a new mount point for the OSS camera directory
2. Create a dedicated xiaomi_cam user
3. Add SMB1 protocol support required by the camera
4. Keep existing nextcloud share intact
5. Use the same container and network setup as your current configuration

## Monitoring and Maintenance

- Monitor available space on SD card
- Monitor ossfs mount status
- Monitor Aliyun OSS usage and costs
- Regular backup verification
- Track storage class transitions
- Monitor lifecycle rule execution
- Verify SMB permissions are maintained
- Monitor failed deletion attempts in Samba logs

## Troubleshooting

Common issues:
- ossfs connection drops
- SMB connection issues
- Storage capacity issues
- Network bandwidth constraints
- Storage class transition delays
- Deep Cold Archive retrieval times (12/48 hours)

## Future Improvements

- Implement monitoring and alerting system
- Optimize storage transition periods
- Consider implementing backup solution
- Add data retrieval automation tools
- Implement video indexing and search capabilities