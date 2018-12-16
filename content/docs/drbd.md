---
title: "Distributed Replicated Block Device (DRBD)"
---

# Distributed Replicated Block Device (DRBD)

**Dec 15, 2018**<!-- \ -->
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

Distributed Replicated Block Device (DRBD) mirrors block devices between multiple hosts. The replication is transparent to other applications on the host systems.

*(Tested on ubuntu:16.04 and ubuntu:18.04)*

# Create a Logical Volume

**References:**

- https://www.digitalocean.com/community/tutorials/how-to-use-lvm-to-manage-storage-devices-on-ubuntu-16-04
- See other post about LVM

It is assumed that there's a volume group named `ubuntu-vg` exists with sufficient free space. You need to create logical volumes on each node.

```bash
lvcreate -L 10G --name drbd-lv ubuntu-vg
```

Our block devices on each node will be `/dev/ubuntu-vg/drbd-lv`.

You should disable the LVM cache in `/etc/lvm/lvm.conf` on both nodes:

```bash
write_cache_state = 0
```

and remove any stale cache entries by:

```bash
sudo rm -rf /etc/lvm/cache/.cache
```

# Install DRBD9 Packages

Add repository in: https://launchpad.net/~linbit/+archive/ubuntu/linbit-drbd9-stack

```bash
sudo add-apt-repository ppa:linbit/linbit-drbd9-stack
sudo apt-get update
```

Install DRBD packages

```bash
sudo apt install -y drbd-utils python-drbdmanage drbd-dkms
```

# Configure DRBD

**References:**

- https://docs.linbit.com/docs/users-guide-9.0/#ch-configure
- https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04
- https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/
- https://www.youtube.com/watch?v=WQGi8Nf0kVc
- https://help.ubuntu.com/lts/serverguide/drbd.html.en
- https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch10.html
- https://www.youtube.com/watch?v=JSP-gk8KHQQ
- https://www.youtube.com/watch?v=qWF13hMbT6g
- https://help.ubuntu.com/community/HighlyAvailableNFS
- http://crmsh.github.io/man-3/
- https://www.sebastien-han.fr/blog/2012/04/30/failover-active-passive-on-nfs-using-pacemaker-and-drbd/
- http://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/
- http://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/pdf/Clusters_from_Scratch/Pacemaker-1.1-Clusters_from_Scratch-en-US.pdf
- http://linux-ha.org/doc/man-pages/man-pages.html
- https://github.com/Azure/azure-quickstart-templates/blob/master/nfs-ha-cluster-ubuntu/scripts/setup_nfs_ha.sh
- https://github.com/ClusterLabs/pacemaker/blob/master/doc/pcs-crmsh-quick-ref.md
- http://manpages.ubuntu.com/manpages/bionic/man8/nfsd.8.html
- https://www.linbit.com/en/persistent-and-replicated-docker-volumes-with-drbd9-and-drbd-manage/


It is assumed that there are two nodes, and their hostnames are `alpha` and `delta`. Our primary node will be `alpha` for this setup.

On each nodes, edit `/etc/hosts` file and add respective hostname and ip addresses. Make sure adding `127.0.1.1` as hostname.

```bash
127.0.1.1       delta
192.168.7.123   alpha
192.168.7.140   delta
```

Select the same timezone on each node, UTC is recommended.

```bash
sudo dpkg-reconfigure tzdata
```

Install `ntp` to synchronize clocks.

```bash
sudo apt-get update
sudo apt-get install -y ntp
```

On each node, edit `/etc/drbd.conf`. Make sure removing other config file includes.

```bash
global {
       usage-count no;
}
common {
       protocol C;
}
resource r0 {
         net {
            fencing resource-only;
         }
         handlers {
            fence-peer "/usr/lib/drbd/crm-fence-peer.9.sh";
            after-resync-target "/usr/lib/drbd/crm-unfence-peer.9.sh";
         }
         on alpha {
            device /dev/drbd0;
            disk /dev/ubuntu-vg/drbd-lv;
            address 192.168.7.123:7788;
            meta-disk internal;
         }
         on delta {
            device /dev/drbd0;
            disk /dev/delta-vg/drbd-lv;
            address 192.168.7.140:7788;
            meta-disk internal;
         }
}
```

Protocol C is synchronous replication protocol. *"Local write operations on the primary node are considered completed only after both the local and the remote disk write(s) have been confirmed."*: https://docs.linbit.com/docs/users-guide-9.0/#s-replication-protocols

Fencing peer is crucial to avoid split brain when the connection between nodes is lost. Handlers ensure that when a node disconnects and falls back from other node, it has to re-sync all the contents from the other node before being primary again.

On each node, load the drbd kernel module, create meta data and bring it online.

```bash
sudo modprobe drbd
sudo drbdadm create-md r0
sudo drbdadm up r0
```

You can check the status with:

```bash
sudo drbdadm status r0
```

You should see something similar to the following.

```bash
r0 role:Secondary
  disk:Inconsistent
  delta role:Secondary
    disk:Inconsistent
```

# Initial synchronization

Initial synchronization has to be started on only one node. Run the following on the primary node - the node selected as the synchronization source.

```bash
sudo drbdadm primary --force r0
```

The synchronization should take a while depending on the logical volume size ~ around 8% per hour for a 500GB HDD volume (PS: It took nearly 13 hours to complete). You should instead start with a small disk and gradually extend the logical volume beneath it as you need more size. You can watch its status by running:

```bash
sudo drbdadm status
```

You should see something similar to the following.

```bash
r0 role:Secondary
  disk:Inconsistent
  alpha role:Primary
    replication:SyncTarget peer-disk:UpToDate done:11.31
```

Once it is complete you will see a status with disks up to date:

```bash
r0 role:Primary
  disk:UpToDate
  delta role:Secondary
    peer-disk:UpToDate
```

Now, only on the primary node, you can format the disk and mount it.

```bash
sudo mkfs.ext4 /dev/drbd0
sudo mount /dev/drbd0 /srv
```

Currently, the data is only served on the primary node, on path `/srv`. If we wanted it to be served on the secondary node, we'd have to unmount it from the primary, and make the drbd resource online and mount it on the secondary manually. To make it highly available, we still need to install and configure a Cluster Resource Manager called `pacemaker`. I will explain it in a different post.
