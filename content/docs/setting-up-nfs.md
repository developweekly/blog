---
title: "Network File System (NFS)"
---

# Network File System (NFS)

**Dec 1, 2018**\
<sup>Last modified: **Dec 2, 2018**</sup>

NFS allows sharing files over a network. By using NFS, users and programs can access files on remote systems almost as if they were local files.

*Tested on ubuntu:16.04*

**References**:

  - https://help.ubuntu.com/community/SettingUpNFSHowTo

# Setting up NFS Server

Run all these commands as `root`.

1.  install nfs-kernel-server:
```bash
apt-get install nfs-kernel-server
```

2.  create the export filesystem:
```bash
mkdir -p /export/mydata
```

3.  bind your data directory `/somedir/mydata` to export filesystem:
```bash
mount --bind /somedir/mydata /export/mydata
```

4.  mount this in `/etc/fstab`:
```bash
/somedir/mydata    /export/mydata   none    bind  0  0
```

5.  export directories in `/etc/exports` to the local network `192.168.7.1/24`:
```bash
/export        192.168.7.1/24(rw,fsid=0,insecure,no_subtree_check,async)
/export/mydata 192.168.7.1/24(rw,nohide,insecure,no_subtree_check,async)
```

6.  restart the service:
```bash
service nfs-kernel-server restart
```

# Setting up NFSv4 Client

Make sure you have an NFS server up and running, see: [nfs-server](#setting-up-nfs-server).

Run all these commands as `root`.

install nfs-common:

```bash
apt-get install nfs-common
```

## Mounting NFS as local file system

- mount root export (defaults to export with fsid=0) to `/mnt`:
```bash
mount -t nfs -o proto=tcp,port=2049 <nfs-server-IP>:/ /mnt
```

- or, mount an exported subtree, to `/home`:
```bash
mount -t nfs -o proto=tcp,port=2049 <nfs-server-IP>:/mydata /home/mydata
```

- to make it automount to `/mnt`, add this to `/etc/fstab`:
```bash
<nfs-server-IP>:/   /mnt   nfs    auto  0  0
```

## Mounting as Docker Volume

A sample `docker-compose.yml`

Docker volume named `nfs` is mounted to container in `/root/nfs`.
```yaml
version: '3.5'
services:
  web:
    volumes:
      - "nfs:/root/nfs"
```
And docker volume `nfs` is mounted with driver options:

  - `driver` is **local**, and its options are similar to `mount` in linux
  - `type` is **nfs**
  - `device` is path of where the exported subtree is in NFS server
  - `proto` is **tcp**, and `port` is **2049** for NFSv4
  - `addr` is IP address of NFS server
  - `rw` is for read/write permissions

```yaml
volumes:
  nfs:
    driver: local
    driver_opts:
      type: nfs
      device: ":/export/mydata"
      o: "addr=192.168.7.140,rw,proto=tcp,port=2049"
```


<script src="https://utteranc.es/client.js"
        repo="developweekly/blog"
        issue-term="title"
        label="comments"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>