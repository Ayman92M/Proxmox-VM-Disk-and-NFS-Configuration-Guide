
# 📄 Proxmox VM Disk and NFS Configuration Guide

This guide provides a detailed, step-by-step approach to setting up a Proxmox VM (referred to here as **the server machine**) with a dedicated disk, configuring an ext4 filesystem within the server machine, and setting up NFS sharing where this VM acts as the **NFS server**. It also includes automated scripts for setting up the NFS server and client.

## 🔧 Prerequisites
- **Proxmox server** with disk configured for the VM
- **Ubuntu or Debian-based distribution** on both the server (the server machine) and client for NFS setup

---

## 📌 Steps

### 1. 💽 Setting Up Disk in Proxmox

1. **Identify the Disk by ID**
   Use the following command on the **Proxmox host** to list the disk IDs and identify the disk you want to add:
   ```bash
   ls -n /dev/disk/by-id
   ```

2. **Assign Disk to VM**
   Use the `qm set` command on the **Proxmox host** to add the identified disk to the server machine (replace VM ID as needed):
   ```bash
   qm set <VM ID> -virtio0 /dev/disk/by-id/THE-DISK-ID
   ```

3. **Restart the VM** (on the Proxmox host)
   ```bash
   qm restart <VM ID>
   ```

   > **Next Step**: After restarting, log into the server machine to list the block devices in the following steps.

---

### 2. 🗄️ Setting Up ext4 Filesystem on Disk (Inside the Server Machine)

> **⚠️ Warning**: The following steps should be performed **inside the server machine**. Formatting with `mkfs.ext4` will erase all data on the specified disk. Double-check that you are using the correct device path.

1. **List Block Devices** (inside **the server machine**)
   Confirm that the disk is available:
   ```bash
   lsblk
   ```
   The output will display all attached storage devices. Look for a device labeled `vda1` or similar, with a size that matches the disk you added in Proxmox. The name `vda1` typically indicates a VirtIO disk on virtualized setups.


2. **Format the Disk** (inside **the server machine**)
   ```bash
   ⚠️ sudo mkfs.ext4 /dev/vda1 ⚠️
   ```

3. **Get UUID for Mounting** (inside **the server machine**)
   Use `blkid` to retrieve the UUID for persistent mounting:
   ```bash
   sudo blkid /dev/vda1
   ```

4. **Edit `/etc/fstab` for Permanent Mount** (inside **the server machine**)
   Open `/etc/fstab` and add the following entry for automatic mounting at boot (replace `abcd-1234` with the actual UUID):
   ```plaintext
   UUID=abcd-1234 /mnt/data ext4 defaults 0 2
   ```

5. **Mount All Filesystems** (inside **the server machine**)
   ```bash
   sudo mount -a
   ```

6. **Verify Mount** (inside **the server machine**)
   Confirm that the mount was successful:
   ```bash
   df -h | grep /mnt/data
   ```

---

### 3. 🌐 Installing and Configuring NFS on the Server Machine

1. **Update and Install NFS Server** (inside **the server machine**)
   ```bash
   sudo apt-get update
   sudo apt install nfs-kernel-server
   ```

2. **Create Shared Directory** (inside **the server machine**)
   ```bash
   sudo mkdir /mnt/myshareddir
   sudo chown nobody:nogroup /mnt/myshareddir
   sudo chmod 777 /mnt/myshareddir
   ```

3. **Configure Exported Shares in `/etc/exports`** (inside **the server machine**)
   Edit `/etc/exports` to specify client IPs or subnet access. Examples:
   - For specific clients:
     ```plaintext
     /mnt/myshareddir {clientIP}(rw,sync,no_subtree_check)
     ```
   - For a subnet:
     ```plaintext
     /mnt/myshareddir {subnetIP}/{subnetMask}(rw,sync,no_subtree_check)
     ```

4. **Export the Shares** (inside **the server machine**)
   Make the shared directory available:
   ```bash
   sudo exportfs -a
   ```

5. **Restart the NFS Kernel Server** (inside **the server machine**)
   ```bash
   sudo systemctl restart nfs-kernel-server
   ```

---

### 4. 🤝 Setting Up NFS on the Client

1. **Install NFS Client** (on the **client machine**)
   ```bash
   sudo apt update
   sudo apt install nfs-common
   ```

2. **Create Mount Point on Client** (on the **client machine**)
   ```bash
   sudo mkdir /mnt/shareddir
   ```

3. **Mount NFS Share Temporarily** (on the **client machine**)
   To test the share, replace `{IP of NFS server}` with the IP address of the server machine:
   ```bash
   sudo mount -t nfs {IP of NFS server}:/mnt/myshareddir /mnt/shareddir
   ```

4. **Configure Permanent Mount in `/etc/fstab`** (on the **client machine**)
   For auto-mounting at boot, add this line to `/etc/fstab`:
   ```plaintext
   {IP of NFS server}:/mnt/myshareddir /mnt/shareddir nfs defaults 0 0
   ```

5. **Mount All Filesystems** (on the **client machine**)
   ```bash
   sudo mount -a
   ```

6. **Verify Mount** (on the **client machine**)
   Confirm the NFS share is mounted:
   ```bash
   df -h | grep /mnt/data
   ```

---

## ⚠️ Important Note on Network Configuration

This NFS setup will work whether the server machine and client are on the same network bridge or different bridges. However, keep the following in mind:

- **Same Network Bridge**: If both server and client are on the same bridge or subnet, communication is straightforward as long as they are within the same IP range. Make sure firewall rules allow NFS traffic (port `2049` for NFS, and sometimes port `111` for portmapper).
- **Different Network Bridges**: If the server and client are on different network bridges or subnets, additional routing or configuration may be required. Ensure that both subnets can pass NFS traffic between them, and configure firewall rules to allow NFS-related traffic.

---

## 📖 Summary
This setup guides you through attaching and formatting a disk on a Proxmox VM (referred to as the **server machine**), setting it up as an NFS server with an ext4 filesystem, and configuring clients to access the NFS share. The included scripts help automate the setup, making it easier to customize the shared directory and client IPs as needed.
