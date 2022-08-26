# Install-Lustre-2.15-with-ZFS-2.1.2-on-Rocky-Linux-8.6

Rocky Linux release 8.6 (Green Obsidian), Kernel: 4.18.0-372.19.1.el8_6.x86_64

## Download Lustre and ZFS rpm packages
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

NB, some develepment rpm packages seem can not be found on Rocky's repositories, even on it's EPEL. For example, the libmount-devel-2.32.1-35.el8.x86_64.rpm 
and  libyaml-devel-0.1.7-5.el8.x86_64.rpm files, I could only found the here:
https://download.rockylinux.org/pub/rocky/8/PowerTools/x86_64/kickstart/Packages/l/
and I had to dwoload them from there manually and install locally.
