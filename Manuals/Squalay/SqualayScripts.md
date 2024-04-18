---
title: Squalay Scripts
description: .. what is it and how to use it
published: true
date: 2024-04-10T13:41:16.415Z
tags: squalay
editor: markdown
dateCreated: 2024-03-08T14:58:19.889Z
---

Squalay, short for **squa**shfs over**lay**, is a deployment system for Linux-based operating systems and
applications. Squalay provides following features:

- Group files by their function (base system, device specific files, application, ...)
- Update/revert entire groups atomically.
    Compared to e.g. apt, where a failed installation leaves the system in an inconsistent state (some files
    present, some not ...) and does not support clean reverts.
- Provide a way to see which files are modified compared to the factory state.
    This also allows to transfer the entire set of modified files to another device - or to group them into a
    customer-specific layer.

# OVERVIEW

Squalay currently has two partitions:
- p1: FAT32, mounted read-only at /mnt/rootfsimg, containing bootloader files, the Linux kernel,
    and the images of the operating system.
- p2: ext4, mounted read-write at /mnt/userdata, containing an area for /etc, /config and
    rwoverlays.

In the system partition, following files are present:
- ./boot.scr contains the bootloader script that loads the Linux kernel and its supporting files.
    Usually, there is no need for customers to change this file. It might be updated as part of MEC
    provided updates.
- ./uEnv.txt contains extra configuration for the bootloader.
    Here, customers can enable additional configuration ("overlays") for the kernel, e.g., to change the
    layout of the backplate connector.
- ./boot.itb contains the Linux kernel and its supporting files (device tree, modules and overlays).
    This file might be updated as part of MEC provided updates.
- ./fstab contains the configuration for the mounting of the rootfs.
    It is used to enable rwoverlays, but also for the initial a/b update support.
- ./ovl-*.img: Those files contain the squalay images.
    Some of them will be provided by MEC, some need to be created by customers.

# LAYERS

## READ-ONLY SQUALAYS

There doesn't seem to be a upper limit on overlayfs lowerdirs, however we advise to keep the number of
layers as small as reasonably possible, while allowing updates and changes to be self-contained and small.

For example, we do not recommend to deploy every single Debian package update as an additional layer on
top of the rootfs-layer, but to update the rootfs-layer when enough updates have been collected. However, in
case of an important issue (e.g. security hotfixes), it might make sense to deploy the single package as a
"hotfix" layer to update the systems. Then, with the next rootfs update, the hotfix overlay can be removed.

Our current design is:

- Rootfs - ovl- 01 - rootfs.img
    Contains the Debian installation with packages we expect to be common in multiple projects.
- Libraries: ovl- 09 - qt5.12.img
    Qt 5.12 Libraries
- Extension of the libraries: ovl- 15 - mec-qml.img
    QML Plugins for hardware access, etc.
- BSP: ovl- 21 - bsp.img
    DM1 specific firmware files, hardware configuration scripts ...
- Application: ovl- 31 - meczero.img
    The MecZero user interface.


An example application could remove the QML plugins, add two layers of its own dependencies and replace
the MecZero application:

- Rootfs - ovl- 01 - rootfs.img
    Contains the Debian installation with packages we expect to be common in multiple projects.
- Libraries: ovl- 09 - qt5.12.img
    Qt 5.12 Libraries
- Extension of the libraries: ovl- 15 - mec-qml.img
    QML Plugins for hardware access, etc.
- BSP: ovl- 21 - bsp.img
    DM1 specific firmware files, hardware configuration scripts ...
- **Application dependencies: ovl- 40 - python3.7.img** Python 3.7 interpreter and modules included
    in the python distribution
- **Application dependencies: ovl- 50 - mypymodule2.1.img** MyPyModule site packages.
- **Application: ovl- 60 - pythonapp.img** Custom Python application

## READ/WRITE-AREAS

- Exposed /mnt/userdata
    The directory /mnt/userdata is mounted as rw by the fstab and is used as storage for /config,
    /etc and for the rwoverlay. This directory is not a layer, but a mountpoint. Thus it hides the content
    of any /mnt/userdata from the squalay. However, the one of the squalays (in our case, the rootfs)
    needs to provide the directory.
- /config
    This directory is bind mounted from /mnt/userdata, for compatiblity with legacy systems. Again, it
    is only a bind-mount and hides the content of /config from any squalay image below and needs to
    be provided from a squalay image.
- /etc
    /etc is a writable overlay over the /etcs from the squalays below. Because of how systemd changes
    the time zone, the entire /etc directory needs to be writable. Because of that, it’s also used to store
    NetworkManager-configuration, the hostname, /etc, without symlinking them to /config. Please see
    the known issue with file modification


## REMOUNTABLE-READ/WRITE

- /mnt/rootfsimg mounts the read-only partition where the images and bootloader configuration is
    stored. To update the device, the command mount -o remount,rw /mnt/rootfsimg must be
    executed. Please see the known issue with updating open files

# WORKFLOWS

## CREATING SQUALAY IMAGES

We provide a script, squalay, to support creating squalay images. Usually, squalay requires a git
repository, because it uses the HEAD short commit id to version the image. The repository is either a "primary
source" (when entire content for the squalay is versioned) or a reference (when the content for the squalay
lies outside the repository, e.g. because it is compiled out of tree).

## OWNERSHIP AND PERMISSIONS

Squashfs supports ownership (whom the file belongs to - root, some system user, ...) and permissions (what
user/group/others can do with the file). However, depending on the source, those information might not be
present or correct. There are two ways to handle the situation:

- -all-root option: if the permissions are correct, but the ownership is not or does not matter, this
    option can be given to squalay to assign all files to root. This works for git repos, as long as the files
    should all belong to root in the final image (git handles permissions but not ownership) and the repo
    does not contain special files (devices/pipes/...)
- perm-git-to-fs/perm-fs-to-git scripts: if the ownerships and permissions are wrong (e.g.
    because the source of the files is a git repository and the files belong to different users), those two
    helper scripts restore/save the ownership and permissions of all files to the files
    .gitperm/.gitslperm for every directory in ./fs.

## IMAGES FROM A REPOSITORY

We recommend the following layout:

~~~bash
./ # root directory of the repo, optionally containing helper scripts
./fs # directory the image should be created from
./fs/etc/... # content for the image
./fs/usr/local/bin/...
~~~
To create an image, run squalay create /output/directory/ovl-XX-myimage.img
/path/to/the/source/repo --gitfs.

## IMAGES FROM RAW SOURCE

We recommend keeping the helper files (systemd service files, scripts, configs, ...) in a fs directory.
~~~bash
./ # root directory of the repo
./fs # directory containing helper scripts
./fs/etc/systemd/system/myapp.service # systemd unit
./src # source code for the application
~~~
If, e.g., your binary is compiled to ./build/myapp, then you will need to copy it into to
./fs/usr/local/bin/ (either inside the repository or after copying fs to a temporary destination) and
run:
~~~bash
squalay create /output/directory/ovl-XX-myapp.img /path/to/the/source/repo
~~~

   - gitfs if you copied myapp to ./fs inside the repo, or:
~~~bash
squalay create /output/directory/ovl-XX-myapp.img /path/to/temporary/fs
~~~
   - /path/to/the/source/repo if you copied ./fs away from the repository.

Please note, that in the first case, you need to create an appropriate .gitignore because otherwise your
squalay image will be marked as dirty.

## IMAGES FROM MULTIPLE REPOSITORIES.

If the source is split across multiple separate repositories (not submodules), we recommend creating separate
squalay images for each of the repositories and merging them together:

~~~bash
squalay create /output/directory/myapp.img /path/to/myapp/repo --gitfs
squalay create /output/directory/myotherapp.img /path/to/myotherapp/repo --gitfs
squalay merge /output/directory/ovl-XX-myapps.img /output/directory/myapp.img
/output/directory/myotherapp.img
~~~

## MODIFYING FILES

Because squashfs is a read-only file system, it is not possible to modify files "in place". Depending on the
change (whether it's a feature/extension/... or a bug fix), it has to be done either in an own layer, or by
editing file in the original repository and re-creating the image.

For bug fixes, we recommend fixing them in the source repository the squalay is created from and the
recreating the image.

For features/extensions, there are two possibilities: modifying files on the workstation or live on the device.

### MODIFYING FILES ON A WORKSTATION

Squalay edit can be used to mount an image with a writeable overlay. For some changes (e.g. installing a
package with apt), a chroot is needed. Other changes (editing plain text config files) can be done without
chroot.

### MODIFYING FILES OR INSTALLING PACKAGES LIVE ON THE DEVICE.

To modify files on the device, it is required to create a "rwoverlay". To do so, you need to:

- Run mount -o remount,rw /mnt/rootfsimg to enable editing on the system partition.
- Modify the file /mnt/rootfsimg/fstab:
- Change the line squalay /mnt/root overlayfs none 0 0 to something like
~~~bash
squalay /mnt/root overlayfs rwoverlay=/mnt/userdata/my_overlay_name 0 0
~~~
   my_overlay_name can be anything matching [a-zA-Z0- 9 - _]*. You can also have multiple
    overlays and switch between them. Your fstab should contain something like:
~~~bash
fit:/mnt/rootfsimg/boot.itb:modules@1 squalay image none 0 0
/mnt/rootfsimg/ squalay imgsource none 0 0
squalay /mnt/root overlay rwoverlay=/mnt/userdata/my_overlay_name 0 0
~~~
- Reboot the device: run reboot

### WORKING WITH RWOVERLAYS

The rwoverlay is persistent across reboots, so any changes to e.g. systemd unit files are available at boot time.
The rwoverlay can be inspected in its directory, i.e. if the option is
rwoverlay=/mnt/userdata/my_overlay_name, then the directory
/mnt/userdata/my_overlay_name/data contains the files that were modified:

- Added files appear as they are.
- Modified files are copies from their original including the modifications.
- Deleted files are special character device files (this removes the file “testfile”)

```bash
root@dm1devkit:/mnt/userdata/rwoverlay/data# ls -lha
total 8,0K
[...]
c--------- 1 root root 0, 0 Jul 23 10:47 testfile
```

You can revert the change of the rwoverlay by deleting the file in the overlays data directory - or revert to
"factory" by deleting everything in the data directory.

Because /etc is an overlay on its own, special care needs to be taken. Depending on the changes
(systemctl enable ..., installed packages, ...), those files must be copied from
/mnt/userdata/etc/data the rwoverlays data/etc (which might need to be created first), or
removed/ignored (e.g. systemds machine-id, ssh server keys, ...).

## REMOVING FEATURES/FUNCTIONS


There are two ways to remove a function, e.g. the MecZero application:

- Disable the service/remove the startup script
    E.g. with systemctl disable \<service>; systemctl mask \<service>, or by removing the
    startup script in /etc/rc.local.d. This is method requires a rwoverlay and it can be used for
    services provided by the rootfs overlay (e.g. NetworkManager). The changes produced here can be
    included in a custom layer that will be created from the rwoverlay.
- Remove the entire layer containing the function
    By removing the entire overlay file (e.g. ovl- 31 - meczero.img), all files belonging to this module
    are removed and do not need to be handled in any custom layer. This method does not require any
    changes in a layer, but can only be used for well-encapsulated functions, e.g. MecZero, BSPd and
    mec-qml.

## FACTORY RESET


To reset the device to factory state:

- remove the rwoverlay from /mnt/userdata/fstab
- remove the rwoverlay directory in /mnt/userdata
- remove changed files or all files in /mnt/userdata/etc
- optionally, depending on the application, remove /mnt/userdata/config

## UPDATING SQUALAY IMAGES ON YOUR DEVICE

By default, squalay images reside in /mnt/rootfsimg (this can be changed in /mnt/rootfsimg/fstab).

Because you can’t just overwrite existing overlays as they are mounted all the time (see Fehler!
Verweisquelle konnte nicht gefunden werden. ) you first have to move the files to another location and
remove them after reboot.

E.g. you receive a new ovl- 01 - rootfs.img the procedure looks like this:
```bash
#make the squalay folder writeable
mount -o remount,rw /mnt/rootfsimg
```
```bash
#create the backup folder
mkdir /mnt/rootfsimg/old
```
```bash
#move any layer you want to replace to the backup folder
mv /mnt/rootfsimg/ovl- 01 - rootfs.img /mnt/rootfsimg/old
```
```bash
#sync written data to disk
sync
```

Now the new squalay image is placed correctly but it cannot be switched on the fly. The system must reboot
to switch to the updated image.

After a reboot you can remove the old layer:

```bash
#make the squalay folder writeable
mount -o remount,rw /mnt/rootfsimg
```
```bash
#remove backup folder
rm -rf /mnt/rootfsimg/old
```
```bash
#sync written data to disk
sync
```
```bash
#make the squalay folder read only
mount -o remount,ro /mnt/rootfsimg
```

You do not need to reboot after deleting the backup folder.

# INTERNALS AND FUNCTIONAL DESCRIPTION

The squalay system is a combination of already existing Linux features squashfs , overlayfs and a customized
initrd/initramfs and fstab.

## SQUASHFS

Squashfs is a compressed, read-only filesystem. It is used to store the directory trees of a squalay image.

Squalay adds metadata into the .squalay directory inside the image. The initramfs mounts those images
and keeps a list of their mount points to be used as lowerdirs in the final overlayfs mount.
The contents of a squashfs image are decompressed "on-the-fly", so that only the image itself needs to be
kept on the eMMC.
## OVERLAYFS

The overlayfs is a Linux filesystem that combines multiple folders as layers on top of each other. It supports a single, optional read-write upperdir and multiple, read-only lowerdirs.

## INITRAMFS

The initramfs is a minimal Linux rootfs that is included in the boot.itb-Image. Its usual purpose is to mount
the real rootfs and then to continue booting.

After the Linux kernel completes its initialization, it executes a single process: /init (initramfs) respectively
/sbin/init (real rootfs). Note: should this process crash, the entire system crashes. In case of the initramfs
/init, it mounts the real rootfs, and then executes switch_root /mnt/root /sbin/init, which
changes the Linux / to the new root and replaces the /init process with /sbin/init.
Our /init is configured by the boot cmdline and handles following:

- Extended fstab for squalay and overlayfs support for /
- Debug shells: shell to drop when loading cmdline, endshell to drop just before boot.
- Export post-cleanup initrd filesystem (i.e. all mount points used in the initrd) via /proc/$(pidof
    forker) for "behind the curtains" access.
- Recovery mode for exporting the mmcblk2 device as an USB mass storage and having a login shell via
    USB.

Please note that sometimes, initrd and initramfs are used interchangeably. While both of them are different
features of the Linux kernel, generally only initramfs is used on modern systems.

## SQUALAY

### EXTENDED FSTAB

The extended fstab is loaded from the root of the partition given in the root= boot argument, in our case
root=/dev/mmcblk2p1. If there is no fstab, the partition is used normally and /sbin/init is executed. If there is a fstab, the fstab is copied away from the partition, the partition is unmounted and the fstab is then parsed
to create /.

All path are as inside the initramfs.

Additional to the usual \<source> \<target> \<fs type> \<options> ... format of the fstab, following
extensions are supported:

- if the \<fs type> is overlay, following extra options are parsed:
    o tmpoverlay\[=\<size>]: Use a temporary upperdir, optionally limit the size.
    o rwoverlay=\<path>: Use a persistent upper dir. Two directories are created in \<path>,
       data, where the overlay is stored, and workdir, an internal directory of overlayfs.
- FIT image sources: \<source> has to be in the format fit:\<path to FIT image>:\<image
    name>
       o \<image name> is extracted from a FIT image in \<path to FIT image> and used as a
          source for mount. The fstype has to be given for the internal image.
       o E.g.: ext4 image ext.img@1 inside the boot.itb:
          fit:/my/path/to/boot.itb:ext.img@1 /my/mount/path ext
          ro,defaults,noatime,nodiratime 0 0
- Squalay workspaces:
    o Squalay workspaces are used to mount a collection of squashfs images as an overlay.
    o A workspace is implicitly created if there is no workspace and something is mounted to the
       squalay target.
          ▪ The \<fstype> parameter is used to distinct between a single image in source
             (image) and a directory of images (imgsource)
    o Mounts can be added to the squalay target and are appended as lowerdir.
    o The squalay source can then be used to mount everything that was added in the workspace
       ▪ The workspace is exited when mounting the squalay source. A new workspace can
          be created afterwards.
       ▪ The \<fs type> field has to be overlay and the supported options are the extended
          overlayfs options described above.

### ANNOTATED EXAMPLES:

- mount every squalay image read-only:

```bash
# mount hardware partition
/dev/mmcblk2p1 /mnt/rootfsimg vfat ro 0 0
# create squalay workspace, load every ovl-*.img from /mnt/rootfsimg.
/mnt/rootfsimg squalay imgsource none 0 0
# mount squalay workspace to /mnt/root
squalay /mnt/root overlayfs none 0 0
```
- include modules-overlay from boot.itb:

```bash
# mount hardware partition
/dev/mmcblk2p1 /mnt/rootfsimg vfat ro 0 0
# create squalay workspace, load every ovl-*.img from /mnt/rootfsimg.
/mnt/rootfsimg squalay imgsource none 0 0
# add modules from boot.itb to the workspace
fit:/mnt/rootfsimg/boot.itb:modules.img@1 squalay image none 0 0
# mount squalay workspace to /mnt/root
squalay /mnt/root overlayfs none 0 0
```

- include read-write partition and mount it as rwoverlay:

```bash
# mount ro hardware partition
/dev/mmcblk2p1 /mnt/rootfsimg vfat ro 0 0
# mount rw hardware partition
/dev/mmcblk2p2 /mnt/userdata ext4 rw,noatime,nodiratime 0 0
# create squalay workspace, load every ovl-*.img from /mnt/rootfsimg.
/mnt/rootfsimg squalay imgsource none 0 0
# add modules from boot.itb to the workspace
fit:/mnt/rootfsimg/boot.itb:modules.img@1 squalay image none 0 0
# mount squalay workspace to /mnt/root
squalay /mnt/root overlayfs rwoverlay=/mnt/userdata/myrwoverlay 0 0
```
- expose rootfsimg and userdata to the runtime root:

```bash
# mount ro hardware partition
/dev/mmcblk2p1 /mnt/rootfsimg vfat ro 0 0
# mount rw hardware partition
/dev/mmcblk2p2 /mnt/userdata ext4 rw,noatime,nodiratime 0 0
```
```bash
# create squalay workspace, load every ovl-*.img from /mnt/rootfsimg.
/mnt/rootfsimg squalay imgsource none 0 0
# add modules from boot.itb to the workspace
fit:/mnt/rootfsimg/boot.itb:modules.img@1 squalay image none 0 0
# mount squalay workspace to /mnt/root
squalay /mnt/root overlayfs rwoverlay=/mnt/userdata/myrwoverlay 0 0
```
```bash
/mnt/rootfsimg /mnt/root/mnt/rootfsimg none bind 0 0
/mnt/userdata /mnt/root/mnt/userdata none bind 0 0
```
# KNOWN ISSUES

## FILES MODIFIED BY MULTIPLE OVERLAYS

Because of how overlayfs handles file overlays (only the uppermost file is visible and changes to a file copies
the entire file to the rw layer), files that might be modified by multiple layers are problematic. Those files are
e.g. /etc/passwd and /etc/shaddow (the user/password database in Linux) and
/var/lib/dpkg/status (the dpkg list of installed packages).

Example:
- "layer 0" as rootfs has its number of packages installed and thus a /var/lib/dpkg/status with
    multiple entries.
- "layer 1", a layer above the rootfs is created that installs the libfoo package. This layer has
    /var/lib/dpkg/status (copied from rootfs) with the modification marking libfoo as installed.

The running system sees the /var/lib/dpkg/status from the "layer 1": The package database contains
libfoo and all packages from "layer 0".

However, "layer 0" is modified to install the libbar package. Its /var/lib/dpkg/status is updated to
reflect that. Now, if the "layer 0" is updated on the device, the booted system will have the files installed by

libbar, but the dpkg database won't have the package marked as "install" because the
/var/lib/dpkg/status is provided by the "layer 1".

To solve this issue, layers above need to be recreated when the layers below modify files that are modified by
both layers.

We plan on solving the issue by providing a framework that can handle file merges of multiple layers (by
concatenating or patching them, or by running custom hooks to merge the files). However, this is currently in
the design stage.

## FILES IN GIT REPOSITORIES

Git only stores plain files, symlinks and their permissions, but not the ownership (user/group), empty
directories or special files (sockets, pipes, char/block devices). Because of this, the missing files and
permissions need to be recreated after a checkout and stored in plain files to be commited.

If the layer is simple (containing only plain files and empty directories containing a .gitkeep file, everything
owned by root) this is not necessary. But more complex layers (e.g. the rootfs layer) require this information
to be restored before being written into a squashfs image.

Currently, this is done by manually executing the ./perm-git-to-fs and ./perm-fs-to-git scripts
contained in the repository. Future squalay implementations might include those scripts.

## UPDATING OPEN FILES

When files are deleted in Linux, they are only removed from the directory structure of the filesystem, the
data itself remains intact and is "garbage collected" when the last process closes the file. Because the squalay
images form the devices rootfs, they cannot be garbage collected until the rootfs itself is unmounted.
However, this currently does not happen. While deleting an image file would work, the space the file
occupied would only be freed when fscking the file system, which also currently does not happen.

While the underlying issue is valid for all files, it only affects the squalay images, because they are mounted.
boot.itb, fstab, boot.scr and uEnv.txt are only open at boot time and then closed, so they can be
updated in-place.

In future, a fully fledged a/b update system will be used so this behavior won't be an issue. For now we
recommend to update the squalay images like described in Updating Squalay images on your device.