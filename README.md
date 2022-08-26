# Install-Lustre-2.15-with-ZFS-2.1.2-on-Rocky-Linux-8.6

Rocky Linux release 8.6 (Green Obsidian), Kernel: 4.18.0-372.19.1.el8_6.x86_64. 

Lustre 2.15.1, Zfs 2.1.2.  For testing purpose, I use the new feature of dRAID(distributed RAID) for building the OST.

#### For my testing purpose, I use loop devices instead of real drives; and I set the MDS/MGS and OSS on the single computer.

## 1 Download Lustre and ZFS rpm packages
Download the Lustre and ZFS rpms from https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/. Belowing are
 the rpm files I downlaoded:

```text
[feng@server1 ~]$ ls Downloads/lustre/
lustre-2.15.1-1.el8.x86_64.rpm  lustre-osd-zfs-mount-2.15.1-1.el8.x86_64.rpm  lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm  

[feng@server1 ~]$ ls Downloads/zfs/
libnvpair3-2.1.2-1.el8.x86_64.rpm  libzpool5-2.1.2-1.el8.x86_64.rpm           zfs-dkms-2.1.2-1.el8.noarch.rpm
libuutil3-2.1.2-1.el8.x86_64.rpm   libzfs5-2.1.2-1.el8.x86_64.rpm             zfs-2.1.2-1.el8.x86_64.rpm
```

#### NB, some develepment rpm packages seem can not be found on Rocky's repositories, even on it's EPEL. 
For example, libmount-devel-2.32.1-35.el8.x86_64.rpm 
and  libyaml-devel-0.1.7-5.el8.x86_64.rpm files, I could only found them here:
https://download.rockylinux.org/pub/rocky/8/PowerTools/x86_64/kickstart/Packages/l/
, and I had to dwoload them from there manually and install locally.

## 2 Install ZFS
For ZFS, I am using the DKMS mode:
```text
[root@server1 lustre]# cd ~/Downloads/zfs/
[root@server1 lustre]# $yum localinstall zfs*.rpm lib*.rpm

[root@server1 lustre]# modeprobe zfs
```
## 3 Install Lustre
For Lustre, I will also use DKMS mode. It seems the Lustre dkms packages have some confilts.  
```text
[root@server1 lustre]# cd ~/Downloads/lustre/
[root@server1 lustre]# yum localinstall lustre-*
Error:
 Problem 1: conflicting requests
  - nothing provides kmod-lustre-osd-zfs needed by lustre-osd-zfs-mount-2.15.1-1.el8.x86_64
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

This is strange, since I am using DKMS mode, so no reason it requires a kmod file. This can only be a mis-configuration of the rpm spec when it was built, and it should be able to be safely ignored. So I went with:

```text
[root@server1 lustre]#rpm  -ivh --node-deps lustre*.rpm
```
The install of the lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm  will report error, while the dkms source and other 2 packages were installed properly.

To properly install lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm, your need to modify 3 files in folder  /usr/src/lustre-zfs-2.15.1/:
(1)dkms.conf, (2)configure, and (3)lustre-dkms_pre-build.sh. Please refer to https://github.com/prod-feng/Install-Lustre2.12.8-on-CentOS-7.9-2009 for more details.

Once the installation of Lustre is done, you can check the dkms status of the zfs and lustre modules:
```text
[root@server1 /]# dkms status
lustre-zfs/2.15.1, 4.18.0-372.19.1.el8_6.x86_64, x86_64: installed
zfs/2.1.2, 4.18.0-372.19.1.el8_6.x86_64, x86_64: installed
```
## 4 Build the ZFS pool using dRAID

```text
###For MDT
[root@server1 /]# for var in {91..92}; do dd if=/dev/zero of=/opt/hd${var}.img bs=1024K count=1024; done

[root@server1 /]# for var in {91..92}; do losetup -o 1048576 /dev/loop${var} /opt/hd${var}.img; done

root@server1 /]# zpool create -O canmount=off -o ashift=12 metapool mirror /dev/loop9[1-2]


###For OST
[root@server1 /]#for var in {21..29}; do dd if=/dev/zero of=/opt/hd${var}.img bs=1024K count=1024; done

[root@server1 /]#for var in {21..29}; do losetup -o 1048576 /dev/loop${var} /opt/hd${var}.img; done

### Builing OST pool uing the new dRAID
[root@server1 /]# zpool create -O canmount=off -o ashift=12 datapool draid1:3d:1s:9c /dev/loop2[1-9]
```

then if everything is fine, you can check the pools ststus:

```text
[feng@server1 ~]$ zpool status
  pool: datapool
 state: ONLINE
config:

	NAME                 STATE     READ WRITE CKSUM
	datapool             ONLINE       0     0     0
	  draid1:3d:9c:1s-0  ONLINE       0     0     0
	    loop21           ONLINE       0     0     0
	    loop22           ONLINE       0     0     0
	    loop23           ONLINE       0     0     0
	    loop24           ONLINE       0     0     0
	    loop25           ONLINE       0     0     0
	    loop26           ONLINE       0     0     0
	    loop27           ONLINE       0     0     0
	    loop28           ONLINE       0     0     0
	    loop29           ONLINE       0     0     0
	spares
	  draid1-0-0         AVAIL   

errors: No known data errors

  pool: metapool
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	metapool    ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    loop91  ONLINE       0     0     0
	    loop92  ONLINE       0     0     0

errors: No known data errors
```

## 5 Set the Lustre storage

```text

[root@server1 /]# mkfs.lustre --reformat --mdt --mgs --backfstype=zfs --fsname=lstore --mgsnode=192.168.0.1 --index=0 metapool/mdt

[root@server1 /]# mkfs.lustre --reformat --ost --backfstype=zfs --fsname=lstore --mgsnode=192.168.0.1 --index=0 datapool/ost

[root@server1 /]# zfs list
NAME           USED  AVAIL     REFER  MOUNTPOINT
datapool      2.01M  5.28G      279K  /datapool
datapool/ost   279K  5.28G      279K  /datapool/ost
metapool       672K   831M       96K  /metapool
metapool/mdt    96K   831M       96K  /metapool/mdt
```
If everything works fine, now to bring up the Lustre storage:

```text
[root@server1 /mkdir -p /mnt/lustre/mylustre-mdt
[root@server1 /mkdir -p /mnt/lustre/mylustre-ost

[root@server1 /mount -t lustre metapool/mdt /mnt/lustre/mylustre-mdt/
[root@server1 /mount -t lustre datapool/ost /mnt/lustre/mylustre-ost/

[root@server1 /]# mkdir /lustre_store
[root@server1 /]# mount.lustre 192.168.0.1@tcp:/lstore /lustre_store
[root@server1 /]# df -h
...
metapool/mdt               816M  7.7M  806M   1% /mnt/lustre/mylustre-mdt
datapool/ost               5.3G   22M  5.3G   1% /mnt/lustre/mylustre-ost
192.168.0.1@tcp:/lstore  5.3G   22M  5.3G   1% /lustre_store
```

