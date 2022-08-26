# Install-Lustre-2.15-with-ZFS-2.1.2-on-Rocky-Linux-8.6

Rocky Linux release 8.6 (Green Obsidian), Kernel: 4.18.0-372.19.1.el8_6.x86_64. 

#### For my testing purpose, I use loop devices instead of real drives; and I set the MDS/MGS and OSS on the single computer.'

## 1 Download Lustre and ZFS rpm packages
Download the Lustre and ZFS rpms from https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/. Belowing are
 the rpm files I downlaoded:

```text
[feng@83-202 ~]$ ls Downloads/lustre/
lustre-2.15.1-1.el8.x86_64.rpm  lustre-osd-zfs-mount-2.15.1-1.el8.x86_64.rpm  lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm  

[feng@83-202 ~]$ ls Downloads/zfs/
libnvpair3-2.1.2-1.el8.x86_64.rpm  libzpool5-2.1.2-1.el8.x86_64.rpm           zfs-dkms-2.1.2-1.el8.noarch.rpm
libuutil3-2.1.2-1.el8.x86_64.rpm   perf-4.18.0-372.9.1.el8_lustre.x86_64.rpm
libzfs5-2.1.2-1.el8.x86_64.rpm     zfs-2.1.2-1.el8.x86_64.rpm
```

#### NB, some develepment rpm packages seem can not be found on Rocky's repositories, even on it's EPEL. 
For example, the libmount-devel-2.32.1-35.el8.x86_64.rpm 
and  libyaml-devel-0.1.7-5.el8.x86_64.rpm files, I could only found the here:
https://download.rockylinux.org/pub/rocky/8/PowerTools/x86_64/kickstart/Packages/l/
and I had to dwoload them from there manually and install locally.

## 2 Install ZFS
Fro ZFS, I am suing the DKMS mode:
```text
cd ~/Downloads/zfs/
yum localinstall zfs*.rpm lib*.rpm

modeprobe zfs
```
## 3 Install Lustre
For Lustre, I will also use DKMS mode. It seems the Lustre dks packages have some confilts. the 
```text
[root@83-202 lustre]# cd ~/Downloads/lustre/
[root@83-202 lustre]# yum localinstall lustre-*
Error:
 Problem 1: conflicting requests
  - nothing provides kmod-lustre-osd-zfs needed by lustre-osd-zfs-mount-2.15.1-1.el8.x86_64
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

This is strange, since the I am using DKMS mode, so no reason it requires a kmod file. This can only be a mis-configuration of the rpm spec when building, and it should be able to be safely ignored. So I went with:

```text
[root@83-202 lustre]#rpm  -ivh --node-deps lustre*.rpm
```
## 4 Set the ZFS pool 

```text
###For MDT
[root@83-202 /]# for var in {91..92}; do dd if=/dev/zero of=/opt/hd${var}.img bs=1024K count=1024; done

[root@83-202 /]# for var in {91..92}; do losetup -o 1048576 /dev/loop${var} /opt/hd${var}.img; done


###For OST
[root@83-202 /]#for var in {21..29}; do dd if=/dev/zero of=/opt/hd${var}.img bs=1024K count=1024; done

[root@83-202 /]#for var in {21..29}; do losetup -o 1048576 /dev/loop${var} /opt/hd${var}.img; done

[root@83-202 /]# zpool create -O canmount=off -o ashift=12 datapool draid1:3d:1s:9c /dev/loop2[1-9]
```
