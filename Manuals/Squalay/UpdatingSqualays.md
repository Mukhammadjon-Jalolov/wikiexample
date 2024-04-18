---
title: Updating Squalay Images
description: This page explains how to update or replace squalay images
published: true
date: 2024-04-10T13:41:42.426Z
tags: squalay
editor: markdown
dateCreated: 2024-03-08T14:33:29.870Z
---

# Pre-Requisits

### Powering the devices

To update the software on the MEC devices, they need to be plugged into a PC or Laptop via the Micro-USB plug in the back of the panel. Be aware that for displays bigger than 4'' the USB connection will need to be actively powered or a powered USB hub might be used.

### Enable USB Update

While plugging in the USB cable, press the button on the bottom of the display panel and hold it for min. 10 seconds. The button itself is on the inside of the display, you can reach it by inserting a needle or something similar in the small hole on the bottom of the device.

After approximately 15 seconds, the device will spawn a USB drive on your PC.


# Image folder

The spawned USB drive already contains different files, which all combined make up the whole system.
Depending on the application, there might be some differences, but the minimum required files in the folder are:

- boot.itb
- boot.scr
- fstab
- uEnv.txt
- ovl-01-rootfs.img
- ovl-09-qt-5.12.img
- ovl-15-mec-qml-5.12.img
- ovl-21-bsp-dm1.img
- ovl-22-libModbusMec.img

There might be additional files,  depending on the application running on the devices.

# Updating images

Every application consists usually of a frontend and a backend component. In the case of the MecZero application, the frontend is **ovl-31-meczero.img** and the backend to it is **ovl-35-bspd.img**. The MecOne application uses **ovl-51-meconeui.img** as its frontend and **ovl-50-mecone-dm1.img** as its backend.

To upload a new application, best practice would be to create a new folder in the USB-exported folder called for example **MecZero**. Now copy the two files mentioned above into the MecZero folder. After that, please copy the new application files provided (eg MecOne) into the USB-exported folder. After doing so, you can now unplug the device and it will automatically start with the new application.

Similarly, if the active application is MecOne, a corresponding folder should be created and the above mentioned files moved into it. 

**Note:**
> It is not mandatory to store the unused applications in the exported folder. To safe storage the image files can of course be stored on the local desktop PC as well.
{.is-info}


### List of images per application

Just be aware that this is a minimum required set of images necessary to run the following applications:

**MecOne**

- boot.itb
- boot.scr
- fstab
- uEnv.txt
- ovl-01-rootfs.img
- ovl-09-qt-5.12.img
- ovl-15-mec-qml-5.12.img
- ovl-21-bsp-dm1.img
- ovl-22-libModbusMec.img
- ovl-29-mCanBridge.img
- ovl-50-mecone-dm1.img
- ovl-51-meconeui.img

**MecZero**

- boot.itb
- boot.scr
- fstab
- uEnv.txt
- ovl-01-rootfs.img
- ovl-09-qt-5.12.img
- ovl-15-mec-qml-5.12.img
- ovl-21-bsp-dm1.img
- ovl-22-libModbusMec.img
- ovl-31-meczero.img
- ovl-35-bspd.img