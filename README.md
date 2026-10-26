# AWS Linux EBS Storage Administration & Persistent Mounting

## Project Overview

This project demonstrates Linux block storage administration on Amazon EC2 using a separately provisioned Amazon Elastic Block Store (EBS) volume. The data volume was attached to an Amazon Linux 2023 instance, formatted with the XFS filesystem, mounted at `/data`, and configured for persistent mounting using a filesystem UUID in `/etc/fstab`.

The lab focuses on operational Linux and AWS storage skills: discovering block devices, creating a filesystem, setting ownership and permissions, validating mounts, and preparing storage to remount after an instance reboot.

## Objectives

- Provision and attach an additional EBS data volume to an EC2 instance.
- Identify the correct Linux block device without modifying the root disk.
- Create an XFS filesystem and mount it at `/data`.
- Set directory ownership and permissions for a non-root user.
- Store and read a test file on the new filesystem.
- Configure a UUID-based persistent mount in `/etc/fstab`.
- Validate the mount configuration and check persistence across a reboot.
- Remove temporary resources after completing the lab to avoid ongoing charges.

## Environment

| Component | Configuration |
| --- | --- |
| AWS Region | `us-east-1` (N. Virginia) |
| VPC | `zen-vpc` |
| Subnet | Private subnet, `10.0.20.0/24` |
| EC2 instance | `zen-linux-ebs-admin` |
| Operating system | Amazon Linux 2023 |
| Instance private IP | `10.0.20.184` |
| Root storage | 8 GiB EBS root volume |
| Additional EBS volume | `zen-linux-data-volume` |
| Data volume configuration | 2 GiB, `gp3`, encrypted at rest |
| Encryption key | AWS managed KMS key: `alias/aws/ebs` |
| Data filesystem | XFS |
| Mount point | `/data` |
| Persistent mounting | `/etc/fstab` entry using filesystem UUID |

> **Device-name note:** The additional EBS volume appeared as `/dev/nvme1n1` in this environment. On EC2 instances using NVMe, Linux device names may differ from the name selected in the AWS console and can change across boots. Always identify the correct volume before formatting it, and use its filesystem UUID for persistent mounting.

## Architecture

The EC2 instance runs inside the private subnet of `zen-vpc` and uses two independent EBS volumes:

- **Root volume:** Contains the Amazon Linux operating system and is mounted at `/`.
- **Data volume:** A separate, encrypted 2 GiB `gp3` EBS volume containing an XFS filesystem mounted at `/data`.

The data volume is attached to the EC2 instance as block storage. Linux uses `/etc/fstab` to locate the filesystem by UUID and mount it at the specified directory during startup.

```text
AWS (us-east-1)
└── zen-vpc
    └── Private subnet (10.0.20.0/24)
        └── EC2: zen-linux-ebs-admin (Amazon Linux 2023)
            ├── Root EBS volume (8 GiB)
            │   └── Mounted at /
            └── zen-linux-data-volume (2 GiB gp3, encrypted)
                └── XFS filesystem → /data
                    └── Persistent mount via UUID in /etc/fstab
```

## Implementation

### 1. Provision and attach the additional EBS volume

In the EC2 console, an additional volume was created with these settings:

- **Name tag:** `zen-linux-data-volume`
- **Volume type:** `gp3`
- **Capacity:** `2 GiB`
- **Availability Zone:** Same Availability Zone as the EC2 instance
- **Encryption:** Enabled with `alias/aws/ebs`

The volume was then attached to `zen-linux-ebs-admin`.

### 2. Identify the new block device

List available block devices and their filesystems:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
lsblk -f
```

The newly attached data volume was identified as `/dev/nvme1n1` in this environment.

**Safety check:** Confirm the intended device is the separate data volume and **not** the operating system's root volume. Filesystem creation overwrites existing filesystem metadata.

### 3. Format the data volume as XFS

```bash
sudo mkfs.xfs /dev/nvme1n1
```

This initializes an XFS filesystem on the **new, empty data volume**. Do not run this command again once data has been written to the volume.

### 4. Create a mount point and mount the filesystem

```bash
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
```

Verify the filesystem type and mount location:

```bash
df -hT /data
lsblk -f
findmnt /data
```

### 5. Configure access permissions and test writes

Allow the default Amazon Linux user to manage files within `/data`:

```bash
sudo chown ec2-user:ec2-user /data
sudo chmod 750 /data
```

Create and read a test file:

```bash
echo "Zen EBS storage lab" > /data/verification.txt
cat /data/verification.txt
```

This checks that the directory is writable by `ec2-user` and that the file can be read back from the mounted filesystem.

### 6. Configure persistent mounting with a filesystem UUID

Retrieve the filesystem UUID:

```bash
sudo blkid /dev/nvme1n1
```

Back up `/etc/fstab` before editing:

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

Capture the UUID in a shell variable:

```bash
UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1)
echo "$UUID"
```

Append the mount entry **once**:

```bash
echo "UUID=$UUID /data xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

Verify the resulting configuration:

```bash
cat /etc/fstab
```

The added line has this format:

```text
UUID=<filesystem-UUID> /data xfs defaults,nofail 0 2
```

**Why these settings matter:**

- **`UUID=`** identifies the filesystem independently of potentially changing NVMe device names.
- **`/data`** is the target mount directory.
- **`xfs`** specifies the filesystem type.
- **`defaults`** applies standard mount options.
- **`nofail`** avoids treating a missing optional data volume as a fatal mount failure during boot.
- **`0 2`** sets the dump and filesystem-check fields; XFS is generally not checked by traditional `fsck` on boot.

### 7. Validate `/etc/fstab` and reboot behavior

Reload systemd's unit configuration and test mounting via `/etc/fstab`:

```bash
sudo systemctl daemon-reload
sudo umount /data
sudo mount -a
findmnt /data
```

`findmnt /data` should show the XFS data filesystem mounted at `/data`.

To perform the persistence test, reboot the instance:

```bash
sudo reboot
```

After reconnecting, verify the mount and the test file:

```bash
lsblk -f
findmnt /data
df -hT /data
cat /data/verification.txt
```

Successful persistence verification means the data filesystem appears at `/data` after startup and `verification.txt` still contains the original test text.

## Verification Checklist

| Check | Command or method | Expected condition |
| --- | --- | --- |
| Separate data disk visible | `lsblk -f` | Additional block device with XFS |
| Filesystem mounted at `/data` | `findmnt /data` | Mount point points to data filesystem |
| Filesystem and capacity visible | `df -hT /data` | XFS with approximately 2 GiB capacity |
| File writable and readable | `cat /data/verification.txt` | `Zen EBS storage lab` |
| Persistent mount entry configured | `cat /etc/fstab` | Single UUID-based `/data` entry |
| Boot persistence | Reboot, then run `findmnt /data` | Data filesystem automatically remounted |

## Troubleshooting Notes

- **`mount` shows usage/help text:** Run `mount` and `df` as separate commands (or separate them with `&&`); do not concatenate command arguments.
- **Device name is different:** Check `lsblk` and match the EBS volume ID; do not assume `/dev/nvme1n1` on every instance.
- **`mount: unknown filesystem type`:** Confirm that the XFS filesystem was created on the correct device.
- **Permission denied when writing to `/data`:** Inspect directory ownership and permissions using `ls -ld /data`.
- **Mount fails after reboot:** Check `sudo blkid`, the UUID in `/etc/fstab`, `sudo mount -a`, and `findmnt /data`. Avoid duplicate `/etc/fstab` entries.

## Cost Management and Cleanup

After completing the lab and retaining any needed documentation:

1. Unmount the data filesystem if the instance is still running: `sudo umount /data`.
2. Terminate the temporary EC2 instance `zen-linux-ebs-admin`.
3. Verify that the 8 GiB root EBS volume was removed if its **Delete on termination** setting was enabled.
4. Find the separate `zen-linux-data-volume`, confirm it is no longer in use, and delete it to stop its ongoing storage charges.
5. Check for any lab-created EBS snapshots or AMIs and remove them if no longer needed.
6. Optionally delete the lab-specific EC2 security group. Security groups do not have a standalone charge.
7. Retain the existing shared `zen-vpc` networking resources for future labs.

> **Cost reminder:** An EBS volume can continue to generate storage charges even after it is detached or after the instance it was attached to is terminated. Deleting a volume is destructive and cannot be undone without a backup or snapshot.

## Skills Demonstrated

- Amazon EC2 and EBS provisioning
- AWS-managed encryption at rest with AWS KMS
- Linux block device discovery (`lsblk`, `blkid`)
- XFS filesystem creation and administration
- Filesystem mounting and mount verification
- Linux ownership and directory permissions
- UUID-based persistent storage configuration using `/etc/fstab`
- Reboot validation, operational troubleshooting, and AWS cost-conscious cleanup

---

**Project focus:** AWS Cloud Operations · Linux System Administration · EBS Storage Management
