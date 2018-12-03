---
title: "Logical Volume Manager (LVM)"
---

# Logical Volume Manager (LVM)

**Dec 4, 2018**<!-- \ -->
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

*"LVM stands for Logical Volume Management. It is a system of managing logical volumes, or filesystems, that is much more advanced and flexible than the traditional method of partitioning a disk into one or more segments and formatting that partition with a filesystem."*

The only way to have a LVM file system on your entire system is a hard format. While installing Ubuntu Server, select partitioning method as *"Guided - use entire disk and set up LVM"*: https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604#7

*(Tested on ubuntu:16.04 and ubuntu:18.04)*

**References:**

- https://www.digitalocean.com/community/tutorials/how-to-use-lvm-to-manage-storage-devices-on-ubuntu-16-04
- https://wiki.ubuntu.com/Lvm
- https://help.ubuntu.com/lts/serverguide/advanced-installation.html.en#lvm
- http://tldp.org/HOWTO/LVM-HOWTO/snapshots_backup.html

You can check your filesystem with:

```bash
$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                                16G     0   16G   0% /dev
tmpfs                              3.2G  1.7M  3.2G   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv 1008G   34G  934G   4% /
tmpfs                               16G     0   16G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                               16G     0   16G   0% /sys/fs/cgroup
/dev/loop0                          89M   89M     0 100% /snap/core/5897
/dev/loop1                          87M   87M     0 100% /snap/core/4917
/dev/loop2                          88M   88M     0 100% /snap/core/5742
/dev/sdb2                          976M  144M  766M  16% /boot
tmpfs                              3.2G     0  3.2G   0% /run/user/1000
```

As seen the filesystem `/dev/mapper/ubuntu--vg-ubuntu--lv` which is mounted on `/`, we have our LVM ready.

# Scan your LVM setup

There are 3 concepts that LVM manages:

- Physical Volumes
- Volume Groups
- Logical Volumes

You can scan your existing physical volumes with

```bash
sudo pvs
```

Similarly, scan volume groups with `vgs` and logical volumes with `lvs`.

# Create or Extend a Volume Group

Upon partitioning with LVM, you would end up with a volume group and a logical volume. Physical volumes are associated with underlying hard drives to be used in volume groups. Check disks with:

```bash
$ sudo lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  88.2M  1 loop /snap/core/5897
loop1                       7:1    0  86.9M  1 loop /snap/core/4917
loop2                       7:2    0  87.9M  1 loop /snap/core/5742
sda                         8:0    0 465.8G  0 disk
├─ubuntu--vg-ubuntu--lv   253:0    0     1T  0 lvm  /
└─ubuntu--vg-drbd--lv     253:2    0 448.1G  0 lvm
  └─drbd0                 147:0    0   448G  0 disk
sdb                         8:16   0 111.8G  0 disk
├─sdb1                      8:17   0     1M  0 part
├─sdb2                      8:18   0     1G  0 part /boot
└─sdb3                      8:19   0 110.8G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0     1T  0 lvm  /
sdc                         8:32   0 931.5G  0 disk
├─ubuntu--vg-ubuntu--lv   253:0    0     1T  0 lvm  /
└─ubuntu--vg-swap_1       253:1    0    36G  0 lvm
```

Here we have `sda`, `sdb`, `sdc` devices. To add a device as physical volume, run:

```bash
sudo pvcreate /dev/sda
```

Once physical volume is created, you can create a volume group with:

```bash
sudo vgcreate ubuntu-vg /dev/sda
```

or expand an existing volume group with:

```bash
sudo vgextend ubuntu-vg /dev/sda
sudo vgextend ubuntu-vg /dev/sdb /dev/sdc
```

You can add as many physical volumes as you like to your volume group. Generally, one volume group per server is sufficient.

# Create or Resize Logical Volume

Logical volumes will be partitions of a volume group. Logical volumes can be extended or downsized within a volume group. To create a logical volume with a `10G` size:

```bash
sudo lvcreate -L 10G --name ubuntu-lv ubuntu-vg
```

or using all the free space remaining on the volume group (You should leave enough free space on volume group though if you are planning to use snapshots, see [snapshots section](#create-snapshots) for more information):

```bash
sudo lvcreate -l 100%FREE --name ubuntu-lv ubuntu-vg
```

To grow the size of a logical volume by `5G`:

```bash
sudo lvresize -L +5G ubuntu-vg/ubuntu-lv
```

To apply the changes, you must also expand the filesystem underneath the logical volume. To handle the filesystem expansion for ext4, run:

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

# Reduce the size of Logical Volume

To reduce the size, you must unmount the filesystem. By booting with a live installation USB medium, you can unmount it (it may already not be mounted):

```bash
sudo umount /dev/ubuntu-vg/ubuntu-lv
```

Check the filesystem to ensure that everything is okay:

```bash
sudo fsck -t ext4 -f /dev/ubuntu-vg/ubuntu-lv
```

To resize the ext4 filesystem with the new size of `15G`:

```bash
sudo resize2fs -p /dev/ubuntu-vg/ubuntu-lv 15G
```

And resize the logical volume with the same size of `15G`:

```bash
sudo lvresize -L 15G /dev/ubuntu-vg/ubuntu-lv
```

Make sure checking the filesystem again with `fsck` command above. If all are fine, you can mount your filesystem, or simply reboot.

# Remove a Physical Volume

If you have enough free space, you can remove a hard drive `sda` without losing any data. This command will move all the data on `sda` to other devices so that it is free and can be safely removed.

```bash
sudo pvmove /dev/sda
```

After all the files are transferred, you can exclude the device from your volume group:

```bash
sudo vgreduce ubuntu-vg /dev/sda
```

And finally remove the physical volume associated with `sda`:

```bash
sudo pvremove /dev/sda
```

You can now remove the disk from your system entirely.

# Create snapshots

Snapshots are just another type of logical volumes, so volume group has to provide free space to create a snapshot. But creating snapshot of a 500G volume doesn't mean you need another 500G of free space. *"It must be large enough to hold all the changes that are likely to happen to the original volume during the lifetime of the snapshot."* At initial construction, snapshots don't hold any space. As the original volume changes, those records are reversed and stored in the snapshot to generate a previous state from the current state of the original volume. If the snapshot logical volume is out of space, it will become unusable and be dropped.

To create a snapshot volume with a size of 100G named `backup_snapshot` of the logical volume `ubuntu-lv`:

```bash
sudo lvcreate -L 100G -s -n backup_snapshot /dev/ubuntu-vg/ubuntu-lv
```

You can now create a mount point and mount your snapshot volume, if you would like to make file operations on it, i.e. taking a backup.

```bash
sudo mkdir -p /mnt/backup_snapshot
sudo mount /dev/ubuntu-vg/backup_snapshot /mnt/backup_snapshot
```

*"You should remove snapshot volume when you have finished with them because they take a copy of all data written to the original volume and this can hurt performance."* To unmount and remove the snapshot:

```bash
sudo umount /mnt/backup_snapshot
sudo lvremove /dev/ubuntu-vg/backup_snapshot
```

This way, you can take backup of a live system without taking the service down or having to worry about file modifications during the backup.
