---
title: "Highly Available NFS Server with Pacemaker & DRBD"
---

# Highly Available NFS Server with Pacemaker & DRBD

**Dec 16, 2018**<!-- \ -->
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

Although containers provide a lot of flexibility, managing the state is not a problem with straight-forward solution on a distributed system. I will help you set up a highly available network file system with replication on multiple hosts. 

We will install NFS Server on both nodes and have Pacemaker control the service, while replication occurs via DRBD (Distributed Replicated Block Device) in the background. You need to have a DRBD resource up and running on both nodes, if not, you can see my other post about installing and configuring DRBD.

*(Tested on ubuntu:18.04)*

# Install & Configure NFS Server

```bash
sudo apt-get install nfs-kernel-server
```

Add our mountpoint on both nodes in `/etc/exports` for network address `192.168.7.0`:

```bash
/srv               192.168.7.0/24(rw,nohide,insecure,no_subtree_check,async)
```

# Install Pacemaker

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

Pacemaker will be our Cluster Resource Manager. First ensure that DRBD service is disabled on both nodes.

```bash
sudo systemctl disable drbd
sudo umount /srv
sudo drbdadm down r0
```

After disabling drbd, install Pacemaker on both nodes.

```bash
sudo apt-get install -y pacemaker
```

On ubuntu 18.04, you may need to install `crmsh` as well.

```bash
sudo apt-get install -y crmsh
```

It is important that you have the same versions of `pacemaker` and `corosync` on all of the nodes, otherwise your cluster will complain about different versions and become unstable.


# Configure Corosync


Now edit `/etc/corosync/corosync.conf` on both nodes. Here most of the configuration comes from the default file, we will just add the following sections.

In `totem` section, add:

```bash
totem {
        ...
        secauth: off
        transport: udpu
        interface {
                ...
                bindnetaddr: 192.168.7.0
                broadcast: yes
        }
}
```

Here `bindnetaddr` is the network address of your nodes. Under `totem`, add a new section `nodelist`.

```bash
nodelist {
         node {
              ring0_addr: 192.168.7.123
              name: alpha
              nodeid: 1
         }
          node {
              ring0_addr: 192.168.7.140
              name: delta
              nodeid: 2
         }
}
```

In `quorum` section, disable `expected_votes: 2` and add the following.

```bash
quorum {
        ...
        #       expected_votes: 2
        two_node: 1
        wait_for_all: 1
        last_man_standing: 1
        auto_tie_breaker: 0
}
```

The final configuration should look like this.

```bash
# Please read the corosync.conf.5 manual page
totem {
        version: 2

        # Corosync itself works without a cluster name, but DLM needs one.
        # The cluster name is also written into the VG metadata of newly
        # created shared LVM volume groups, if lvmlockd uses DLM locking.
        # It is also used for computing mcastaddr, unless overridden below.
        cluster_name: debian

        # How long before declaring a token lost (ms)
        token: 3000

        # How many token retransmits before forming a new configuration
        token_retransmits_before_loss_const: 10

        # Limit generated nodeids to 31-bits (positive signed integers)
        clear_node_high_bit: yes

        # crypto_cipher and crypto_hash: Used for mutual node authentication.
        # If you choose to enable this, then do remember to create a shared
        # secret with "corosync-keygen".
        # enabling crypto_cipher, requires also enabling of crypto_hash.
        # crypto_cipher and crypto_hash should be used instead of deprecated
        # secauth parameter.

        # Valid values for crypto_cipher are none (no encryption), aes256, aes192,
        # aes128 and  3des. Enabling crypto_cipher, requires also enabling of
        # crypto_hash.
        crypto_cipher: none

        # Valid values for crypto_hash are  none  (no  authentication),  md5,  sha1,
        # sha256, sha384 and sha512.
        crypto_hash: none

        # Optionally assign a fixed node id (integer)
        # nodeid: 1234

        secauth: off
        transport: udpu

        # interface: define at least one interface to communicate
        # over. If you define more than one interface stanza, you must
        # also set rrp_mode.
        interface {
                # Rings must be consecutively numbered, starting at 0.
                ringnumber: 0
                # This is normally the *network* address of the
                # interface to bind to. This ensures that you can use
                # identical instances of this configuration file
                # across all your cluster nodes, without having to
                # modify this option.
                bindnetaddr: 192.168.7.0
                # However, if you have multiple physical network
                # interfaces configured for the same subnet, then the
                # network address alone is not sufficient to identify
                # the interface Corosync should bind to. In that case,
                # configure the *host* address of the interface
                # instead:
                # bindnetaddr: 192.168.1.1
                # When selecting a multicast address, consider RFC
                # 2365 (which, among other things, specifies that
                # 239.255.x.x addresses are left to the discretion of
                # the network administrator). Do not reuse multicast
                # addresses across multiple Corosync clusters sharing
                # the same network.
                # mcastaddr: 239.255.1.1
                # Corosync uses the port you specify here for UDP
                # messaging, and also the immediately preceding
                # port. Thus if you set this to 5405, Corosync sends
                # messages over UDP ports 5405 and 5404.
                mcastport: 5405
                # Time-to-live for cluster communication packets. The
                # number of hops (routers) that this ring will allow
                # itself to pass. Note that multicast routing must be
                # specifically enabled on most network routers.
                ttl: 1
                broadcast: yes
        }
}

nodelist {
         node {
              ring0_addr: 192.168.7.123
              name: alpha
              nodeid: 1
         }
          node {
              ring0_addr: 192.168.7.140
              name: delta
              nodeid: 2
         }
}

logging {
        # Log the source file and line where messages are being
        # generated. When in doubt, leave off. Potentially useful for
        # debugging.
        fileline: off
        # Log to standard error. When in doubt, set to no. Useful when
        # running in the foreground (when invoking "corosync -f")
        to_stderr: no
        # Log to a log file. When set to "no", the "logfile" option
        # must not be set.
        to_logfile: no
        #logfile: /var/log/corosync/corosync.log
        # Log to the system log daemon. When in doubt, set to yes.
        to_syslog: yes
        # Log with syslog facility daemon.
        syslog_facility: daemon
        # Log debug messages (very verbose). When in doubt, leave off.
        debug: off
        # Log messages with time stamps. When in doubt, set to on
        # (unless you are only logging to syslog, where double
        # timestamps can be annoying).
        timestamp: on
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}

quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
        #       expected_votes: 2
        two_node: 1
        wait_for_all: 1
        last_man_standing: 1
        auto_tie_breaker: 0
}
```

Make sure you have the same exact configuration on both nodes. Now on both nodes, restart `corosync` and start `pacemaker`.

```bash
sudo systemctl restart corosync
sudo systemctl start pacemaker
```

You can see that 2 nodes are configured by running:

```bash
sudo crm status
```

There are currently no configured resources on the cluster.


# Add Cluster Resources


Now we will create cluster resources. But before that, let me explain what this configuration does.

First, we want our drbd resources to be up with a primary/secondary configuration, and we are favoring the primary to be `alpha`.

```bash
primitive drbd_nfs ocf:linbit:drbd \
	params drbd_resource=r0 \
	op monitor interval=15s
ms ms_drbd_nfs drbd_nfs \
	meta master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
location loc ms_drbd_nfs 100: alpha
```

Then, we want to mount our drbd device to `/srv`.

```bash
primitive fs_nfs Filesystem \
	params device="/dev/drbd0" directory="/srv" fstype=ext4 options="noatime,nodiratime" \
	op start interval=0 timeout=60 \
	op stop interval=0 timeout=120
```

And we want to serve this directory with an nfs server:


```bash
primitive nfs ocf:heartbeat:nfsserver \
	params nfs_init_script="/etc/init.d/nfs-kernel-server" nfs_shared_infodir="/srv/nfs_shared" \
        nfs_ip=nfs.mydomain.com \
	op monitor interval=5s
```

In order for NFS daemons to keep track of client sessions/locks after a failover, we must use a shared folder within our drbd filesystem. *"The nfsserver resource agent will save nfs related information in this specific directory. And this directory must be able to fail-over before nfsserver itself."* I created `/srv/nfs_shared` folder for this purpose, and configured it as `nfs_shared_infodir` as seen above.

When you shutdown the primary node, we want pacemaker to make sure secondary node is the new primary and serving your data on `/srv` path. We now have a highly available DRBD cluster, but we don't know where the current primary is at a given time.

We can solve this by adding a virtual IP service on the cluster. But I prefer routing NFS traffic with a load balancer, since I already have HAProxy configured. NFS clients will connect to the balancer on `nfs.mydomain.com`, and load balancer will do health checks on the NFS servers behind it. Since only one NFS server will be up at a given time, load balancer will route the traffic to it. This way we've added another layer so that we are able to read/write this data with a smooth failover.

Rest of the configuration makes sure that the services start with a desired order and colocated on the primary node.

To enter `crm` interactive prompt, type the following on one of the nodes.

```bash
sudo crm configure
```

You should see `crm(live)configure#`. Now add the following configurations line by line.

```bash
property stonith-enabled=false
property no-quorum-policy=ignore
primitive drbd_nfs ocf:linbit:drbd \
	params drbd_resource=r0 \
	op monitor interval=15s
primitive fs_nfs Filesystem \
	params device="/dev/drbd0" directory="/srv" fstype=ext4 options="noatime,nodiratime" \
	op start interval=0 timeout=60 \
	op stop interval=0 timeout=120
primitive nfs nfsserver \
	params nfs_init_script="/etc/init.d/nfs-kernel-server" nfs_shared_infodir="/srv/nfs_shared" \
        nfs_ip=nfs.mydomain.com \
	op monitor interval=5s
group HA fs_nfs nfs \
	meta target-role=Started
ms ms_drbd_nfs drbd_nfs \
	meta master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
order fs-nfs-before-nfs inf: fs_nfs:start nfs:start
location loc ms_drbd_nfs 100: alpha
order ms-drbd-nfs-before-fs-nfs inf: ms_drbd_nfs:promote fs_nfs:start
colocation ms-drbd-nfs-with-ha inf: ms_drbd_nfs:Master HA
show
commit
quit
```

Running `show` should display the following configuration.

```bash
node 1: alpha \
	attributes standby=off
node 2: delta \
	attributes standby=off
primitive drbd_nfs ocf:linbit:drbd \
	params drbd_resource=r0 \
	op monitor interval=15s
primitive fs_nfs Filesystem \
	params device="/dev/drbd0" directory="/srv/drbd" fstype=ext4 options="noatime,nodiratime" \
	op start interval=0 timeout=60 \
	op stop interval=0 timeout=120
primitive nfs nfsserver \
	params nfs_init_script="/etc/init.d/nfs-kernel-server" nfs_shared_infodir="/srv/drbd/nfs_shared" \
        nfs_ip=nfs.mydomain.com \
	op monitor interval=5s
group HA fs_nfs nfs \
	meta target-role=Started
ms ms_drbd_nfs drbd_nfs \
	meta master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
order fs-nfs-before-nfs inf: fs_nfs:start nfs:start
location loc ms_drbd_nfs 100: alpha
order ms-drbd-nfs-before-fs-nfs inf: ms_drbd_nfs:promote fs_nfs:start
colocation ms-drbd-nfs-with-ha inf: ms_drbd_nfs:Master HA
property cib-bootstrap-options: \
	have-watchdog=false \
	dc-version=1.1.18-2b07d5c5a9 \
	cluster-infrastructure=corosync \
	cluster-name=debian \
	stonith-enabled=false \
	no-quorum-policy=ignore
```

Finally, running `sudo crm status` you should see your configured nodes and resources.

```bash
Stack: corosync
Current DC: delta (version 1.1.14-70404b0) - partition with quorum
Last updated: Mon Dec 10 13:53:21 2018
Last change: Mon Dec 10 13:40:23 2018 by root via cibadmin on delta

2 nodes configured
4 resources configured

Online: [ alpha delta ]

Full list of resources:

 Master/Slave Set: ms_drbd_nfs [drbd_nfs]
     Masters: [ alpha ]
     Slaves: [ delta ]
 Resource Group: HA
     fs_nfs	(ocf::heartbeat:Filesystem):	Started alpha
     nfs	(lsb:nfs-kernel-server):	Started alpha
```


# Load Balancing NFS Servers


And my load balancer configuration in `/etc/haproxy/haproxy.cfg` on `nfs.mydomain.com` is as follows:

```bash
frontend nfs
    bind *:2049
    mode tcp
    option tcplog
    log /dev/log local0
    # tcp-request connection accept if { src -f /etc/haproxy/whitelist.txt }
    default_backend nfs_backend

backend nfs_backend
    mode tcp
    option tcplog
    server nfs_backend_alpha 192.168.7.123:2049 check
    server nfs_backend_delta 192.168.7.140:2049 check backup
```

The problem with NFS daemon on ubuntu:16.04 is that you can't set `lease-time` and `grace-time` parameters. Lease time corresponds to how often clients need to confirm their state with the server. Similarly, new file requests will not be allowed until grace time has passed. These settings help prevent clients from getting stale file and io errors when a failover happens. The clients will experience some delay in io operations (depending on the parameters) but they will eventually be able to complete their io operations.  As far as I know, grace time for nfsd on ubuntu:16.04 is 90 seconds. This is why I had to upgrade servers to ubuntu:18.04, in order to set a shorter period. You may need to manually configure `/etc/default/nfs-kernel-server` and `/etc/init.d/nfs-kernel-server` to be able to apply changes on nfsd and check `/proc/fs/nfsd/nfsv4gracetime` and `/proc/fs/nfsd/nfsv4leasetime` files to make sure they are configured correctly. Similarly, you can change the default port of nfsd as well. (see more on nfsd man pages)


# Testing Failover Manually


First, install nfs client if you haven't already:

```bash
sudo apt-get install nfs-common
```

Now, mount nfs:

```bash
sudo mount -t nfs -o proto=tcp,port=2049 nfs.mydomain.com:/srv /mnt
```

Now we will test a dummy write operation to `/mnt`:

```bash
sudo dd if=/dev/zero of=/mnt/dummy count=50 bs=1M
```

This should be completed fairly shortly. Now let's drain our primary node and try to write on the other node without re-mounting our nfs drive. On primary node `alpha`, run:

```
sudo crm node standby alpha
```

This should give you a status of the following when checked `crm status`:

```bash
Node alpha: standby
Online: [ delta ]

Full list of resources:

 Master/Slave Set: ms_drbd_nfs [drbd_nfs]
     Masters: [ delta ]
     Stopped: [ alpha ]
 Resource Group: HA
     fs_nfs	(ocf::heartbeat:Filesystem):	Started delta
     nfs	(ocf::heartbeat:nfsserver):	Started delta
```

Now our primary is `delta`. We will now test a second write in our nfs client.

```
sudo dd if=/dev/zero of=/mnt/dummy2 count=50 bs=1M
```

This should also be completed within a similar period.

To bring back node online, you should run:

```
sudo crm node online alpha
```

You can now test the same scenario, but with a larger file. And before the write is completed, drain the primary node and let nfs do a failover. Your write operations should be a bit slower, but completed without any errors. You should be able to see all dummy files on both nodes (as long as the node is primary), and on your nfs client.

If you experience any hick up, you can umount with `sudo umount -l /mnt` and try mounting again. Sometimes the lag between switching two nfs servers is longer than the grace period parameter you've set.

If I were you, I would test this setup for a while before migrating all my data to it. It is looks like a nice setup but may not be as performant as you expect under heavy network loads. I am planning to move the state of a Docker Swarm Cluster to a HA-NFS cluster like this and mount docker volumes via nfs but I am still concerned about the performance and agility of an automatic failing over. Still, it was a nice exercise for me to understand `corosync` and `pacemaker` as well as tweaking with `nfsd` on different versions of ubuntu. 