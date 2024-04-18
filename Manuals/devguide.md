---
title: Developer Guide
description: 
published: true
date: 2024-04-10T13:40:20.391Z
tags: 
editor: markdown
dateCreated: 2024-03-29T06:52:11.258Z
---

# MEC Developer Guide

## 1 Getting started

To be able to access the device remotely via ssh and to modify/add data on the device please follow the steps below:

### **1.1 Power up the device**

Connect the device to a Micro-USB power supply (please also note the comments [here](#h-2-hardware)). The device will start automatically into the MecZero application, that enables you to setup the device for development. This will include ssh access and preparing the device for read-write access.

### **1.2 Connecting to WiFi**

In the MecZero application navigate to the *WiFi* section on the left menubar. You will be presented with the available **access points**. Choose the desired access point and enter the connection settings. After being connected return to the WiFi menu. By clicking on the the info icon on the top bar you can determine the **IP address** of the device.

### **1.3 Enable ssh access**

Select the System section on the left menubar and then choose the SSH tab for **SSH configuration**. Check **SSH enabled** (to have ssh enabled already on system start) and **SSH active** (will activate ssh for the current session). Select **Yes** for **Permit root use** (allows the root access with password).

### **1.4 Enable the read-write overlay**

You are now ready to **connect to the device** via SSH. To do so, in a terminal write:

      ssh root@<IP address>

The password is **1** or – on some devices – **dm1devkit.** 

To get **write permission** on the filesystem it is necessary to edit the file */mnt/rootfsimg/fstab*. To do so, first remount the partition mounted on */mnt/rootfsimg* to be able to modify it by executing

      *mount -o remount,rw,async /mnt/rootfsimg* 

Next, in an editor of your choice (like nano, vi, …) open the file */mnt/rootfsimg/fstab* and comment out the line… 

```plaintext
#squalay /mnt/root overlay none 0 0
```

and comment in the line… 

```plaintext
squalay /mnt/root overlay rwoverlay=/mnt/rw/myrwoverlay 0 0
```

Afterwards execute

*mount -o remount,ro /mnt/rootfsimg* 

and reboot the system via

      *reboot*

### **1.5 Deploy personal ssh key**

Generate an **SSH keypair** on your local machine: 

ssh-keygen -f <key-filename> -C <human readable comment, e.g. your email address>

After generating the keys deploy **your public key** to the device by executing the command: 

*$> ssh-copy-id -i root@ <path to public\_key> root@<IP address>*

Answer “yes” to the question if you want to continue connecting. Again, you will be prompted to enter the password.

## **2 Hardware**

MEC devices are based on Linux with an ARM (Cortex A7) CPU (Allwinner A20). Thus, applications need to be cross compiled from the host to the ARM target architecture.

The device can be powered via micro-USB. Please make sure that the supply delivers at least constant(!!!) **1000mA.** The bigger the screen size the bigger the power consumption, so the supply must be adjusted to avoid unwanted behavior of the device.

### **2.1 Connector**

20-Pin Connector interface to backplate:

![](/connector_screen.png)
Connector card

|     |     |     |     |     |
| --- | --- | --- | --- | --- |
| **Fixed** | **PIN** | **PIN** | **Current assignment** | **GPIO** |
| N/C | **2** | **1** | UART1 (RTS) | PI16 |
| +5Vin | **4** | **3** | N/A | PG8, PH20 |
| USB\_N | **6** | **5** | N/A | PH21 |
| USB\_P | **8** | **7** | UART0 (TX) | PI12, PB21 |
| GND | **10** | **9** | UART0 (RX) | PI13, PB20 |
| ETH\_3V3 | **12** | **11** | GPIO | PG6 |
| ETH\_TX\_N | **14** | **13** | GPIO | PI17, PG7 |
| ETH\_TX\_P | **16** | **15** | UART1 (TX) | PI18 |
| ETH\_RX\_N | **18** | **17** | UART1 (RX) | PI19 |
| ETH\_RX\_P | **20** | **19** | GND (Fixed!) |     |


### **2.2 Interfaces**

The interface mapping depends on the deployed Linux-image and can be re-mapped on request by MEC. 

By default, the following interfaces are available:

| **Type** | **Name** | **Interface** |
| --- | --- | --- |
| SPI | SPI1 | *(not mapped by default)* |
| UART | UART-0<br><br>UART-1<br><br>UART-2 | **/dev/uart-0** or **/dev/uart-backpic**<br><br>**/dev/uart-1** or **/dev/uart-backplate**<br><br>(not mapped by default) |
| CAN | CAN1 | (not mapped by default) |
| I2C | I2C2 | (not mapped by default)<br><br>[https://www.kernel.org/doc/Documentation/i2c/dev-interface](https://www.kernel.org/doc/Documentation/i2c/dev-interface) |
| GPIO |     | (see GPIO chapter) |

On older Linux images the UART interfaces 0 and 1 are named **/dev/uart-backpic** and **/dev/uart-backplate** respectively. Please check which notation is present in your case.

**_Important:_**

When the device is used with a MEC backplate (rather den a custom developed one):

The **UART-0** interface is claimed by the system to communicate with the backplate’s PIC and it is not possible to be used. 

The **UART-1/UART-2** interfaces can be used to communicate with the “outer world" (RS485), e.g. Modbus.

### **2.3 GPIO**

The available GPIOs are listed in **/sys/class/gpio/\*** and again depend on the hardware mapping in the Linux image (“device tree”) and can be adapted by MEC on demand.

A detailed documentation can be found here: [https://www.kernel.org/doc/Documentation/gpio/sysfs.txt](https://www.kernel.org/doc/Documentation/gpio/sysfs.txt)

Writing a value to OUT-GPIO:

echo 1 > /sys/class/gpio/gpio274/value # set to high

Reading a value from IN-GPIO :

cat /sys/class/gpio/gpio274/value # read GPIO state

**Calculation of sysfs gpio number:**

Each GPIO has a number in the sysfs tree. To calculate the GPIO number to the correct PIN from the 20-pin connector the following formula is used: **x\*32+y**, whereas **“x”** is the character (0-based index) and **“y”** is the number from the 20-pin connector table (see above). 

For example: **PI18** => I=8 => 8\*32+18 = **274**.

**Important:**

In case of running BSPd on the device the **GPIOs 198 (PG6) and 199 (PG7)** are currently claimed by the application and shouldn’t be written to. 

### **2.4 Sensors**

**2.4.1 BSP/BSPd**

The device’s sensors should be read via the BSP/BSPd interface provided by MEC. The BSPd provides filesystem access to sensor properties and values. BSPd by default mounts to **/run/bspd/\*** (or **/tmp/mec/\*** on older system versions).

A QML plugin (QML import url: “Mec.BSP”) for easy access to BSP data is available as well. Please see the separate documentation for BSPd and/or the MEC QML plugins for more information.

**2.4.2 Backlight**

On systems without a provided BSPd the display backlight can be controlled via **/sys/class/backlight/\***

Detailed information can be found here:

[https://www.kernel.org/doc/Documentation/ABI/stable/sysfs-class-backlight](https://www.kernel.org/doc/Documentation/ABI/stable/sysfs-class-backlight)

### **2.5 eMMC**

**2.5.1 Liftetime**

[https://elinux.org/images/2/2c/Wear\_Estimation\_for\_Devices\_with\_eMMC\_Flash\_Memory.pdf](https://elinux.org/images/2/2c/Wear_Estimation_for_Devices_with_eMMC_Flash_Memory.pdf)

The eMMC has a finite number of write/erase cycles. After that it will enter an end-of-life read-only state. Therefore, it is important to limit the number of write cycles and be aware of everything that writes onto the eMMC. 

The eMMC offers the option to create hardware partitions and then converting these partitions into an enhanced mode. This normally switches the MultiLevelCells into pseudoSingleLevelCells, reducing the capacity but increasing the lifetime. 

Most of these eMMC options are just one-time programmable.

## **3 System**

### **3.1 Connect the Device to a WiFi Network**

To remotely access the MEC device it must be connected to a network (WiFi in most cases).

In case there is no default UI deployed on the device (see section 3.2 about **MecZero UI**BSPd Interface Documentation) this must be done via API or via serial connection directly to the device. 

**3.1.1 API**

There is also a QML API available. This API is provided via the **Mec.System** plugin and can be imported in any QML application. It is also possible to link against the plugin from a C++ application. The plugins are located in the Qt path (**/usr/lib/mec-qt//qml/Mec/…**)

The plugin is constantly extended, so for the latest version please contact Mec.

**3.1.2 Serial Connection**

Connect the serial cable to the connector on the device’s board as seen in the following picture. Make sure the GND wire (black) is connected to the square connector! Next to GND please connect your RXD wire (middle connector), followed by the TXD wire.

![](/serial_connection.png)

To access the device’s terminal via the serial cable you can use PuTTY with the following settings:

![](/putty_settings.png)

Set “Serial line” to the COM port the system assigned when connecting the cable to PC. The “speed” (Baudrate) should be set to **115200**.

When the cable is connected and the settings have been specified click the “Open” button and a terminal window will open. Hit the ENTER key and a prompt should appear which requests a username and password.

When entering the correct credentials Linux shell commands can be entered.

To get the current IP address, use the **“ifconfig”** or **“nmcli”** command.

To connect to a new WiFi network, execute the **“nmtui”** command and follow the screen options to connect to a WiFi network.

### **3.2 MecZero UI**

MecZero is an initially deployed UI on Mec devices to configure the system which for example lets you connect to a LAN/Wifi network in order to start developing on the device.

|     |     |
| --- | --- |
| ![](/ethernetconninfo.png) | **Ethernet screen**<br><br>![](/ethernetlogo.png)<br><br>The icon in the top right indicates if the device is connected to a wired LAN network by color turning green (connected) or red (disconnected).<br><br>Currently connecting to a LAN network is only possible via DHCP. Static configuration will follow in future.<br><br>If connected the screen presents the current connection infos like the assigned IP address, subnet mask, gateway and nameservers. |
| ![](/wifiscreen.png) | **Wifi screen**<br><br>![](/wifiicon.png)<br><br>The icon in the top right indicates if the device is connected to a wired LAN network by color turning green (connected) or red (disconnected). The switch en-/disables WiFi functionality.<br><br>-   Shows the current connection infos, similar to the ethernet screen.<br><br>The screen shows a list of all stored WiFi connections and available WiFi access points to connect to.<br><br>If connected to a WiFi network the entry is colored green. |
| ![](/sensors.png) | **Sensors**<br><br>This screen lists all available sensors on the device and displays the corresponding value it delivers.<br><br>Some sensors provide extra information (like the TOF/distance sensor). This is indicated by the icon next to the value. A click/tap on the value brings you to the sensor’s extra screen. |
| ![](/extrainfosensor.png) | The extra information screen for the TOF/distance sensor shows an example of configured regions and visualizes the measured values in the regions. |
| ![](/wifisettings.png) | **Wifi settings screen**<br><br>This screen shows the WiFi connection properties like the WiFi password (if encrypted) and the connection mode (DHCP/Static).<br><br>A click on saves and updates the connection properties.<br><br>**Note:** Editing the current active connection will trigger a reconnection. |
| ![](/systemtimezone.png) | **System Time & Timezone screen**<br><br>Enables setting of the system’s local time – if NTP (automatic time synchronization via dedicated internet services) is disabled. <br><br>The timezone can be selected from a list by clicking the button.<br><br>The icon ( or ) indicates if the current system time is synchronized via NTP. |
| ![](/systemscreen.png) | **System screen**<br><br>Provides various system settings and actions. <br><br>• Reboot the system <br><br>• Change the default password (of the root user) <br><br>• Change the hostname <br><br>• Basic SSH configuration <br><br>• Activate USB modes <br><br>• Display disk usage information <br><br>• System information <br><br>• and many more |
| ![](/prodmode.png) | **Production Mode**<br><br>Conveniently provides a way to download a script and execute it on the device. This can be very time saving if you need to update a bigger batch of devices.<br><br>Note: The chosen wifi connection will only be used temporarily and won’t be saved across device boots.<br><br>The default values displayed on the page can be set via the MecZero config file (/etc/mec/meczero.conf) <br><br>A press on the start button connects to the WiFi and tries to download the chosen script and execute it. The output of the script is shown on the next page. |
| ![](/sshconfig.png) | **SSH configuration screen**<br><br>Enables basic SSH configuration.<br><br>An **enabled** SSH service is automatically started on system boot. An **active** service is currently running.<br><br>The **port** on which the SSH service should listen to (22 is the default).<br><br>**Root access** can be disabled or permitted using a key file.<br><br>After changing settings make sure to restart the service (e.g. by deactivating and activating it again) |
| ![](/usbscreen.png) | **USB screen**<br><br>Lets you activate special USB modes on the MicroUSB port (backside of the device). The modes are mutual exclusive.<br><br>The **serial** mode opens a USB serial port in the system (on **/dev/ttyGS0**). It’s also possible to launch a **terminal** interpreter on the port (connection settings are the same as described in the serial connection section).<br><br>The **mass storage** mode can expose an image file (FAT filesystem) or an emulated (read-only) CDROM drive (ISO image). The FAT image can be written by the mounting system.<br><br>To create such an (optionally writeable) FAT image [see this example](https://linux-sunxi.org/USB_Gadget/Mass_storage#Preparing_shared_storage_device). |
| ![](/systeminfo.png) | **System information screen**<br><br>Shows information about the operating system, hardware, graphics system and qt.<br><br>For example CPU frequency, memory, OpenGL version, etc. |

**3.2.1 Disable MecZero UI**

To disable MecZero from being executed at system start-up a file must be edited on device.

On Debian buster devices the system service must be disabled:

```plaintext
systemctl disable meczero
```

### **3.3 Splash Screen**

The Splash Screen is the image shown at device startup. The image used for the splash screen is replaceable. It needs to be a Bitmap (.bmp) format with a bit depth of up to 24-Bit. An image with a larger bit depth cannot be displayed. To check the bit depth of an image file on windows, right click the file and select Properties. A new window will open, select the tab Details.

By default, there are four image files on the device:

• splash\_480x480.bmp - This image is for 4-inch devices with 480px \* 480px resolution. 

• splash\_1024x600.bmp - This image is for 7-inch devices with 1024px \* 600px resolution. 

• splash\_1280x800.bmp - This image is for 7-inch devices with 1280px \* 800px resolution. 

• splash.bmp - This is the fallback image if none of the above images exists for the respective devices. 

The images are located in the ‘splash’ directory found on the first partition of the eMMC(mounted as readonly at **/mnt/rootfsimg/**). It is recommended that the image used has the same pixel size as the screen. This is because the image is not scaled to fit on the screen E.g. a small pixel sized image will be positioned centrally and will not be scaled to fit the entire screen.

There are 2 options to test a different splash image.

1.  Replace the corresponding default image file in **/mnt/rootfsimg/splash/.**
2.  Use a custom image with a different file name. If this option is chosen, a Kernel command line argument should be used to specify the image file name. Simply add **splash=<image>** into **/mnt/rootfsimg/uEnv.txt** file after the **addBootArgs** parameter. As an example, if the image file is named **custom\_splash.bmp** the kernel command line argument should be **splash=custom\_splash**

**NOTE:** For option 2, the custom image should be rotated by 180 degrees for the 4inch devices before being saved.

**3.3.1 Change the Default Splash Image With SSH**

Using the SSH tool, login to the device and remount the first partition of the eMMC as read-write. This can be done with the following command: 

```plaintext
mount -o remount,rw /mnt/rootfsimg/
```

Afterwards, place your image file in /mnt/rootfsimg/splash/ and confirm that the splash parameter in /mnt/rootfsimg/uEnv.txt matches the image file name. remount the partition again as read-only and reboot:

```plaintext
mount -o remount,ro /mnt/rootfsimg/
```

The new splash image should be shown on the next boot cycle.

### 3.4 Factory Reset

DM1 Debian Buster devices which have the squalay layer mechanism deployed can be reset to factory state. For this just all overlay data can be deleted to revert to the initial state taken completely from the deployed squalay images: 

```plaintext
rm -rf /mnt/userdata/myrwoverlay/data/*
```

```plaintext
rm -rf /mnt/userdata/etc/data/*
```

## **4 Qt**

For UI/application development Qt is deployed on the device. Available backends are X11 (software rasterized) and EGLFS (hardware accelerated), whereas EGLFS is the recommended default. For compilation GCC is used

### **4.1 Qt Application Bootstrapping**

To run Qt applications with EGLFS (OpenGL ES) backend and QtWebEngine some bootstrapping is necessary.

Environment variables can either be set by specifying them in *“/etc/environment*” file globally on the device or also via C++ code (e.g. in the very beginning of the main() implementation via qputenv()).

**4.1.1 Qt EGLFS Backend**

To enable GPU accelerated graphics using OpenGL ES2 the Qt EGLFS platform backend must be set by specifying the following environment variables: 

**• QT\_QPA\_EGLFS\_INTEGRATION=eglfs\_mali**

**• QT\_QPA\_PLATFORM=eglfs**

**4.1.2 Qt WebEngine**

If you want to use QtWebEngine module (incl. WebGL support) in your application following changes are required:

_Call parameters for your application:_

**• --ignore-gpu-blacklist** 

**• --egl** 

**• --disable-es3-gl-context** 

**• --use-gl=egl** 

**• --enable-checker-imaging** 

**• --num-raster-threads=3** 

**• --enable-gpu-rasterization** 

**• --enable-native-gpu-memory-buffers** 

**• --touch-events=enabled**

      o This is needed to have proper touch event support in JavaScript/DOM. Without this flag it seems event handlers for touch events can be registered on DOM elements. But the DOM elements themselves are missing the handlers (for example “ontouchstart”) which some JS libraries assume to be present when checking for touch support.

**• --enable-zero-copy**

      o The --enabled-zero-copy parameter causes some websites (e.g. google.com) to be rendered incorrectly, so it’s recommended to not add this parameter for now

Alternatively, all those parameters can also be set via the **QTWEBENGINE\_CHROMIUM\_FLAGS** environment variable. E.g.

```plaintext
export QTWEBENGINE_CHROMIUM_FLAGS=”--ignore-gpu-blacklist --usegl=egl ...”
```

**_IMPORTANT:_** Since mecqt-5.12 (Chromium 69) the flag **„--no-sandbox“** (or alternatively the environment variable **QTWEBENGINE\_DISABLE\_SANDBOX=1**) is mandatory for QtWebEngine to run. This is because the application runs as root user in the system. Without this flag the application prints a warning and quits as soon as a QtWebEngine view is displayed.

The warning will look like the following:

```plaintext
[26891:26891:0930/105424.497942:ERROR:zygote_host_impl_linux. cc(89)] Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
```

_Initialization code in main() method:_

```cpp
#if defined(QT_WEBENGINE_LIB)
# include <QtWebEngine>
#endif

int main(int argc, char *argv[])
{
 QGuiApplication app(argc, argv);
#if defined(QT_WEBENGINE_LIB)
 // initialize the OpenGL context (needed for HW acceleration)
 QtWebEngine::initialize();
#else
#warning "No QtWebEngine support"
#endif
 ...
 return app.exec();
}
```

**4.1.2.1 QtWebEngine Debugging and Profiling**

QtWebEngine offers to enable [Developer Tools](https://doc.qt.io/qt-5/qtwebengine-debugging.html). Those developer tools let you inspect and change the DOM tree and also the applied CSS of the current website. Further it lets you inspect the requests made by the site and do some performance analysis and much more.

To enable access to those developer tools you need to either set the **QTWEBENGINE\_REMOTE\_DEBUGGING** environment variable or add the **\--remote-debugging-port** startup parameter to your application with a value like **0.0.0.0:8888** (Note that 0.0.0.0 is important to be able to connect to the dev tools remotely – on any network interface).

```plaintext
export QTWEBENGINE_REMOTE_DEBUGGING=0.0.0.0:8888
```

Once the dev tools are enabled and a QtWebEngine page is active in the application you can use any Chrome/Chromium based browser on your Desktop machine to connect to dev tools running on the device, by simply browsing to the device’s IP address on the specified port.

![](/chromedesktop.png)

Note: Since Chrome v80 (Desktop) the following call parameters may be needed to display the inspect view, for example:

```plaintext
"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" --enableblink-features=ShadowDOMV0 --enable-blink-features=CustomElementsV0
```

**4.1.2.2 Chromium URLs**

QtWebEngine supports some ***chrome://*** urls to inspect some internal data/infos of WebEngine. Probably the most important one might be ***chrome://gpu*** to check graphic acceleration capabilities.

Some supported urls are:

|     |     |
| --- | --- |
| chrome://appcache-internals | Information about appcached sites, including how much space they use |
| chrome://blob-internals | Information about Binary Large Objects (blobs) |
| chrome://gpu | Information about hardware acceleration and supported features |
| chrome://histograms | Histogram related information |
| chrome://indexeddb-internals | IndexedDB information in the user profile |
| chrome://media-internals | Displays media information when playing media |
| chrome://network-errors | Displays a list of network error messages that might be thrown by the engine |
| chrome://serviceworker-internals | Lists all Service Workers registered by the engine and options to unregister |
| chrome://webrtc-internals | Create a dump by downloading PeerConnection updates and stats data |

| **Debugging urls** |     |
| --- | --- |
| chrome://crash | Simulates a crash |
| chrome://kill | Kills the current page and displays a "killed" page instead |
| chrome://hang | Simulates a „frozen“ browser |
| chrome://gpuclean |     |
| chrome://gpucrash | Simulates a crash of the gpu |
| chrome://gpuhang | Simulates a „frozen“ gpu |
| chrome://ppapiflashcrash | Simulates a crash of PPAPI Flash |
| chrome://ppapiflashhang | Simulates a hang of PPAPI Flash |

### **4.2 Mec QML Plugins**

The device already has some QML plugins (provided by Mec) deployed which deliver some useful utility QML types for easier development of UIs and other functionality on the device.

Those plugins are either placed in in the Qt path (**/usr/lib/mec-qt/5.12/qml/Mec/\***). In (the rare) case they are placed in “/usr/bin” the QML import path (“/usr/bin” in this case) must be [added to the QQmlEngine](https://doc.qt.io/qt-5/qtqml-syntax-imports.html#qml-import-path).

**4.2.1 Documentation**

The toolchain package comes with a HTML and QtCreator documentation which shows what QML types the plugins provide and how to use them.

Extract the **mec-qml-docs.zip** archive to a location of your choice. 

To integrate the documentation directly into QtCreator and be able to show the documentation of an element directly in code add the **.qch** file to QtCreator.

In QtCreator open **_Tools → Options → Help_** and add a “Documentation”: 

![](/mecqml.png)

**4.2.2 C++ Linking**

The Mec QML plugins can also be linked against in C++/Qt. This requires adding the include path and the libraries to the compiler building the project. In QtCreator/qmake this can be done as the following: 

`INCLUDEPATH += "$${PWD}/include"`

`LIBS += -L"$${PWD}/libs/Mec/System" –lMecSystem`

`QMAKE_LFLAGS += " -Wl,-rpath,/usr/lib/mec-qt/5.12/qml/Mec/System "`

The `QMAKE_LFLAGS` variable containing the **rpath** parameter telling the linker (during runtime on the target device) where to look for the plugin/shared-object file. So we set it to the default location inside the Qt directory.

Most implementations inside the plugins require an event loop running and/or the existence of a QApplication/QGuiApplication. Thus the `main()` method implementation should look something like the following: 

```plaintext
#include <Mec/MecGlobal.h>
#include <Mec/System/WifiListModel.h>
int main(int argc, char** argv)
{
QApplication app(argc,argv);
WifiListModel wifiModel;
//...
return app.exec();
}
```

### **4.3 Performance Hints**

There are some hints when developing Qt/QML UIs on the device using the EGLFS backend. 

It is important to keep in mind that a higher CPU load means a higher power consumption and thus an immediate increase of the device’s temperature. Therefore, it is advised to avoid unnecessary operations on the CPU. 

Beside of code optimizations there are other options to tune Qt internal painting and avoid unnecessary CPU load, which have distinctly impact when using animations in the UI. 

The **QT\_QPA\_EGLFS\_FORCEVSYNC=1** environment variable forces Qt’s EGLFS backend to avoid unnecessary painting of frames. 

When setting the environment variable **QSG\_RENDER\_LOOP=basic**, Qt’s render loop is executed in the main/UI thread, which eliminates the need for thread synchronization to the dedicated render thread. 

[https://doc.qt.io/qt-5/qtquick-visualcanvas-scenegraph-renderer.html#performance](https://doc.qt.io/qt-5/qtquick-visualcanvas-scenegraph-renderer.html#performance)

## **5 Developing**

### **5.1 Preparing**

**5.1.1 Use 3rd Party Libraries / dev-packages**

To use 3rd party libraries, install dev-packages (either on the device directly or inside a docker image) and transfer the needed libs and header files locally into your project and [compile/link against them](https://doc.qt.io/qt-5/third-party-libraries.html). 

To run a dev environment within Docker MEC provides also mecqt images: 

```plaintext
docker pull mecelectronics/qt-armhf-buster:5.12.5
docker run --rm -v ${pwd}:/out --entrypoint=/bin/bash
mecelectronics/qt-armhf-buster:5.12.5
```

This launches a Linux environment using docker with a mecqt-5.12 image. In there you can install Linux development packages and copy the needed files to the mounted directory “/out”:

```plaintext
apt update
apt install somepkg-dev:armhf
tar chvf /out/somepkg.tar /usr/lib/somepkg*.so* /usr/include/somepkg
```

**_Note:_** The h-parameter of the tar command is needed to duplicate symlinks of the target file, since Windows doesn’t support symlinks out-of-the-box. 

**5.1.2 Toolchain Build Detection**

Often it is useful to detect if the current build process is (cross-)building for the MEC device or not, e.g. when you are testing the UI on the Windows host for rapid development. 

Since mecqt-5.12 the toolchain provides QMAKE variables and compiler macros for distinction.

In the qmake .pro file you can use **mecToolchain** or **xBuild** conditions. 

```plaintext
mecToolchain {
 SOURCES += myLinuxSource.cpp
}
```

In C/C++ you can use the **MEC\_TOOLCHAIN\_BUILD** and **\_XBUILD** preprocessor macros.

### 5.2 Toolchain

Building applications can either be done with the MEC toolchain described in this section or alternatively with docker images provided by MEC.

For a list of docker images please see the available tags matching the Qt version deployed on the device. 

Debian Buster: [https://hub.docker.com/r/mecelectronics/qt-armhf-buster](https://hub.docker.com/r/mecelectronics/qt-armhf-buster)

Example:

```plaintext
docker pull mecelectronics/qt-armhf-buster:5.12.5 
docker run --rm -v ${pwd}:/source -v ${pwd}/out:/out mecelectronics/qtqt-armhf-buster:5.12.5 MyApp.pro –-strip
```

After a successful build, the binary is copied to the „out“ directory. The default entrypoint of the docker images support **.pro** and **CMakeLists.txt** files.

**5.2.1 Installation**

Extract the MEC toolchain. The path must not contain any spaces.

e.g. **C:\\Qt\\mecqt-5.12**

**5.2.1.1 Requirements**

• min. QtCreator v7.0 

• Recommended: git for Windows (used for git-bash and ssh executables) 

**5.2.2 QtCreator Setup**

The following instructions were created using **QtCreator 7.0**. The screenshots may vary for older/newer versions.

**5.2.2.1 Add Remote Device**

First, we need to specify a remote MEC device where we want to deploy the application to and debug it remotely. 

In QtCreator open **_Tools → Options → Devices_** and add a new **“Generic Linux Device”.**

![](/addremotedevice.png)

Most of the fields can be left filled with their default values. 

The most needed fields are: 

-   **Name**

      The name of this device configuration which is also used to identify it later 

-   **Authentication type**

      It is advised to use key authentication for convenience reasons. Otherwise, each single command execution will ask for the user password.

-   **Private key file**

      The local path to the private key file which is used for authentication of the ssh connection to the device. With a click on the **“Create new”** button you can create a development private/public key pair. You can also deploy the public key directly to the device with the **“Deploy Public Key”** button on the right side.

      _Note:_ to deploy the key to the device you must set the “Authentication type” to “default” (password authentication) to successfully transfer the key to the device. The device also must be writeable (which it is not by default). To do so please refer to [the read-write overlay](#read-write-overlay).

-   **Host name**

      the hostname or IP address of the MEC device to connect to

-   **GDB server executable (optional)**

      The (remote) path to the gdbserver executable on the MEC device. In case a custom compiled gdbserver version is deployed on the device you can specify it here. If left empty the default installed gdbserver executable is looked up.

If all settings have been made you can test the connection to the device with a click on the **“Test”** button (on the right side).

**5.2.2.2 SSH/SFTP Setup**

Since QtCreator 4.9 you can explicitly specify the executables for establishing the SSH/SFTP connection:

![](/ssh_setup.png)

To add at least executables for SSH & SFTP you can use your local *“Git for Windows”* installation if you have one installed already. When the directory **“C:\\Program Files\\Git\\cmd”** is in the **PATH** environment variable QtCreator will pick up the executables automatically.

It is also possible to install *OpenSSH* to provide the executables.

For QtCreator version 4.8 or lower make sure that executables for *ssh* and *sftp* can be found in the PATH environment variable.

**5.2.2.3 Add Debugger**

To enable remote debugging on the MEC device we need to specify a GDB debugger executable. 

In QtCreator open **_Tools → Options → Kits_** and open the **“Debuggers”** tab.

To manually add a debugger, click the “Add” button on the right side panel.

![](/add_debuggers.png)

Give it a meaningful name and browse for the gdb.exe in the previously extracted MEC toolchain.

E.g. **“C:\\Qt\\mecqt-5.12\\arm-mec-linux-gnueabihf\\bin\\arm-mec-linux-gnueabihf-gdb.exe”**

**5.2.2.4 Add Compilers**

To cross compile applications for the MEC device we need to add gcc/g++ cross compilers to QtCreator.

In QtCreator open **_Tools → Options → Kits_** and open the **“Compilers”** tab.

![](/add_compilers.png)

To manually add a compiler, click the **“Add”** button on the right-side panel and add a **GCC** compiler for C and C++.

The path to the gcc and g++ executables must be taken from inside the previously extracted toolchain. E.g. **“C:\\Qt\\mecqt-5.12\\arm-mec-linux-gnueabihf\\bin\\arm-linux-gnueabihf-gcc.exe”**

The ABI should be selected automatically (if not specify it like in the screenshot above) and shouldn’t be changed to avoid warnings in the configuration.

**5.2.2.5 Add Qt Version**

Add a cross compile Qt version (identical to the version on the MEC device) to QtCreator.

In QtCreator open **_Tools → Options → Kits_** and open the **“Qt Versions”** tab.

Add the mecqt setup by clicking the **“Add”** button on the right-side panel and selecting its **qmake.exe.**

E.g. **“C:\\Qt\\mecqt-5.12\\bin\\qmake.exe”** 

![](/add_qt_version.png)

**5.2.2.6 Add Kit**

Finally add a Kit which then can be used to build a project in QtCreator.

In QtCreator open **_Tools → Options → Kits_** and open the **“Kits”** tab. 

Click the “Add” button on the right-side panel.

![](/add_kit.png)

Now all the previous added components of the build process (Device, Compiler, Debugger, Qt version) need to be “bundled” in this Kit.

The most important options are

-   **Name**

The name of the Kit which is used to identify it later when assigning it to the project. If you like you can add the MEC icon (contained in the toolchains root directory) to the Kit to identify the Kit quickly. 

-   **Device Type**

Select “Generic Linux Device”

-   **Device**

From the list select the device created in the previous steps

-   **Sysroot**

Leave empty! Is automatically set by the toolchain

-   **Compiler**

Select the gcc and g++ created in the previous steps

-   **Debugger**

Select the GDB debugger created in the previous steps

-   **Qt version**

Select the Qt version created in the previous steps

**5.2.3 Add Kit to Project**

To assign a previously created Kit to a project to build, deploy and debug the created application, open the project in QtCreator and select the “Projects” settings (on the left side panel).  
By simply selecting, the Kit it is enabled for this project. Greyed out Kits are disabled for the build of this project.  
 

**5.2.3.1 Build settings**

In order to get the debugger also working on QML-files disable the Qt Quick Compiler:

![](/build_settings.png)

**To enable the Qt Quick Compiler for release-builds** add these lines to your projects.pro file:

```plaintext
CONFIG(release, debug|release) {
      CONFIG *= qtquickcompiler
}
```

**5.2.3.2 Run Settings**

The run settings give control of the deployment process to the device and how the application is executed. 

![](/run_settings.png)

![](/run_settings_2.png)

The files to deploy and the target directories are taken from the projects .pro file.

The following lines specifiy where the target executable of this project should be deployed (**“/tmp”** in this case). If you create a project with QtCreator’s wizard, you may find lines like the following which define the entries in the **“Files to deploy”** box: 

`# Default rules for deployment.`

`qnx: target.path = /tmp/$${TARGET}/bin`

`else: unix:!android: target.path = /tmp`

`!isEmpty(target.path): INSTALLS += target`

For further infos read: [https://doc.qt.io/qtcreator/creator-deployment-embedded-linux.html](https://doc.qt.io/qtcreator/creator-deployment-embedded-linux.html)

In the “Debugger Settings” section you can en-/disable C++ and/or QML debugging.

### 5.3 Deployment to the Device

By default, MEC devices are mounted read-only. This is necessary to keep the system in a valid state after an unexpected power interrupt.

To have write access to the device please refer to the [read-write overlay](#read-write-overlay).

**_Note:_** To list all write-mounted partitions you can use the command (**“mount | grep rw”**).

**5.3.1 Run Application on Device Startup**

There are different techniques to automatically run an application on device startup. After you are done with development and want to make the application automatically appear upon device startup you can do the following (while being connected to the device via ssh – or even with WinScp):

Mount the device writeable like described above.

Create a shell script (e.g. **/usr/local/bin/runUI.sh**) which runs the UI executable („/usr/local/bin/myGuiApp”). The following script sets up all necessary environment variables needed by Qt and executes the application. 

```plaintext
#!/bin/bash
export QT_QPA_EGLFS_INTEGRATION=eglfs_mali
export QT_QPA_PLATFORM=eglfs

/usr/local/bin/myGuiApp
```

**5.3.1.1 systemd Service**

Using systemd services gives more control and a cleaner integration into the system. To add the application as a systemd service place a file (e.g. “myui.service”) in **/etc/systemd/system** with the following contents: 

```plaintext
[Unit]
Description=MyUI
After=network.target
StartLimitIntervalSec=0 

[Service] 
Type=simple
Restart=always
RestartSec=1
ExecStart=/usr/local/bin/runUI.sh

[Install]
WantedBy=multi-user.target
```

Now to start the application automatically on device startup the unit must be “enabled”. This can be done with

```plaintext
systemctl enable myui.service
```

or by simply placing a symlink to the .service file in **/etc/systemd/system/multi-user.target.wants**

```plaintext
ln -s /etc/systemd/system/myui.service /etc/systemd/system/multiuser.target.wants/myui.service
```

**5.3.1.2 View Log Output**

To see the log output of an application running as a system service you can run journalctl command:

`journalctl -u myui -f`

**5.3.1.3 Useful systemctl Commands**

To interact and control the systemd service the systemctl command can be used in the following way

`systemctl <CMD> myui.service`

| <CMD> |     |
| --- | --- |
| start | start the unit, if not already running |
| restart | stop and/or (re-)start the unit |
| stop | stop the unit |
| enable | starts the unit on system boot (doesn’t start the unit) |
| disable | removes auto start on system boot (doesn’t stop the unit) |

### 5.4 Custom Docker Images

It is possible to create custom docker images based on the docker images provided by MEC (https://hub.docker.com/u/mecelectronics), for example when you are using additional packages on the  device.)), for example when you are using additional packages on the device. 

A minimal Dockerfile could look like this:

```plaintext
FROM mecelectronics/qt-armhf-buster:5.12.5
RUN apt update && apt install -y additionalPackage-dev:armhf && apt clean
```

Build the image by executing the following command in the folder of the Dockerfile:

```plaintext
docker build -t mycompanyrepo/myimagename:myversiontag .
```

### 5.5 Cross Building Python Modules

For MEC images containing Python it might be necessary to install additional modules sooner or later. Some Python modules require to compile code locally during the installation process. Since the devices by default do not have compilers and dev-packages installed (for storage reasons), such modules need to be crosscompiled (to the ARM target architecture).

This could be done theoretically on the device itself by installing the compilers, but for time/performance/storage reasons it is recommended to do this on a Desktop machine and transfer the modules to the target device.

**5.5.1 Docker Container**

To cross compile Python modules, a Docker image containing the necessary environment is provided by MEC. The environment contains a preconfigured **Python-crossenv** ([https://pypi.org/project/crossenv/](https://pypi.org/project/crossenv/)) setup.

Python (x86\_64) is located in **/usr/bin/python3.7**, whereas the target Python (armhf) is located in **/usr/local/bin/python3.7**. The crossenv virtual environment is initialized in **/root/venv**. 

To see the list of available containers/tags please take a look at the [mecelectronics docker-hub repository](https://hub.docker.com/r/mecelectronics/python-crossenv).

**5.5.1.1 Starting Container**

The following command starts a “one-shot” container (the instance is removed after the container is exited).

```plaintext
docker pull mecelectronics/mecelectronics/python-crossenv:3.7.8_buster
docker run -it --rm mecelectronics/mecelectronics/python-crossenv:3.7.8_buster 
```

To start a named container:

```plaintext
docker run -it --name python_env mecelectronics/mecelectronics/python-crossenv:3.7.8_buster
```

Continue the named container:

```plaintext
docker start -a -i python_env
```

**5.5.2 Building Python Modules**

To cross-build python modules you first must enter the virtual environment:

```plaintext
$ source /root/python-crossenv.sh
```

The terminal lines are now prefixed with “(*cross*)”. If you encounter link errors, try “`python-crossenvcc.sh`” which explicitly sets the compiler to the cross compiler (this seems to be an internal issue in crossenv). You can now start installing Python modules by simply calling pip:

```plaintext
$ (cross) pip -v install numpy
```

To exit the virtual environment call

```plaintext
$ (cross) deactivate
```

**5.5.3 Transfer Built Modules to Device**

To transfer the built modules to the device you can call:

```plaintext
$ /root/python-deploy.sh <device-ip>
```

### 5.6 Building Embedded Wizard projects

To build Embedded Wizard projects for the platform MEC devices are running on it is necessary to use the appropriate *Build Environment*, to modify the *Makefile* and to build the project finally using a docker image provided by MEC, which has the needed cross-compilers, OpenGL graphic libraries and header-files already installed.

The necessary Build Environment is **IMX7-OpenGL-fbdev** and can be downloaded [here](https://doc.embedded-wizard.de/getting-started-imx7-opengl-fbdev?v=11.00) from the Embedded Wizard website. After downloading the Build Environment and generating the code with it, modify the **CFLAGS** variable in the *Makefile* to look like the following:

`CFLAGS = -O2 -Wall -march=armv7-a -mtune=cortex-a7 -mfpu=neon -DLINUX=1 \ -DMESA_EGL_NO_X11_HEADERS=1 -mfloat-abi=hard`

After doing so please use the *mecelectronics/cc-buster-armhf* docker image to build the binary.

This can be done like so:

\- Start the container interactively and mount the working directory to /source:

***docker run -it --entrypoint /bin/bash -v <path\_to\_build\_environment>:/source mecelectronics/ccbuster-armhf:latest***

\- In the container move to /source/Application/Project and build the Embedded Wizard binary:

***ARCH=arm CROSS\_COMPILE=arm-linux-gnueabihf- CC=arm-linux-gnueabihf-gcc make***

\- After successfully building, the resulting binary will be placed next to the *Makefile* in the *IMX7-OpenGL-fbdev/Application/Project* directory. It is now ready to be uploaded onto the device and to be executed.

### 5.7 Debugging

**5.7.1 Debugging Aids**

To get information about actual rendering speed shown in the application output console, you can set `QSG_RENDER_TIMING = 1`

**5.7.2 Debug Symbols**

To properly debug applications remotely on the device GDB needs to load debug information - if available.

By default, GDB tries to load the debug symbols directly from the device. Which is a painfully slow and has to be done for every debug session again. Also, there is just not enough space available on the device to store all the debug symbols.

No actions are needed if you just want to debug the application itself, since the application is compiled with embedded debug symbols (in debug mode only). But when you want to step into certain libraries (like Qt libs for example) some additional bootstrapping is needed.

**5.7.3 Deploy gdbserver**

To enable remote debugging on the device **gdbserver** must be installed on the device. Since this is ok for development devices it is discouraged for productive devices (for security reasons).

So either install gdbserver via the command 

`apt-get install gdbserver`

or deploy it along with the build process as needed. This way the *gdbserver* executable is deployed to the “/tmp” directory on the device and automatically deleted when the device is rebooted. To do so add the following line to your qmake project file:

`INSTALLS += mec-gdbserver`

Make sure to specify the path to the gdbserver executable (“**/tmp/gdbserver**”) in the device settings of QtCreator’s settings dialog.

To change the target path (“/tmp” by default) you can do the following:

`mec-gdbserver.path = /config`

`INSTALLS += mec-gdbserver`

**_Note:_**

For QtCreator versions **<4.10** an additional build step is needed to make the file executable. In the project  

settings add the following custom command deploy step:

`chmod +x /tmp/gdbserver`

This is done automatically when using QtCreator 4.10 onwards.

![](/deploygdbserver.png)

**5.7.4 Transfer Libraries from Device**

Since mecqt-5.12 the recommended way to link against 3rd party (dev-)libraries is as follows:

There is a script called `copy-sysroot-from-device.sh` to update the toolchain’s sysroot directory (if needed). (You most probably do not ever need to call this script.) 

**5.7.5 Extract Qt Debug Symbols**

The debug symbols of the Qt libs are contained in an archive found in the root of the MEC toolchain.

Extract the contents of the archive to “<base-path>/usr/lib” – where <base-path> is the path you executed the `copy-sysroot-from-device.sh` script in

In the end make sure you have file paths like:

**<base-path>/usr/lib/debug/usr/lib/mec-qt/5.12/lib/libQt5DBus.so.5.12.5.debug**

**5.7.6 GDB Setup**

Now to let GDB load the libs and debug symbols from the local machine rather than remotely from the device we set the **sysroot** parameter to the path to the directory where you loaded the libs from the device (using the `copy-sysroot-from-device.sh` script).

In QtCreator go to **_Tools → Options → Debugger_** and open the “**GDB**” tab. Now insert the *sysroot* and *debug-file-directory* commands in the “**Additional Startup Commands**” field:

![](/gdb_setup.png)

**5.7.7 Source File Mapping**

To properly step through Qt libraries, you also need to tell GDB where to find the Qt source files referenced in the libraries/debug-symbols.

Download the Qt source files (e.g. from https://download.qt.io/archive/qt/5.12/5.12.5/single/) and extract  them to a folder of your choice.  Make sure the Qt version matches the Qt version used by the toolchain!)) and extract them to a folder of your choice. 

Make sure the Qt version matches the Qt version used by the toolchain!

In QtCreator go to **_Tools → Options → Debugger_** and open the “**General**” tab. In the “*Source Paths mapping*” group press the “Add” button.

In the “Source Path” field enter “**/qtsrc/qt/qt-everywhere-src-5.12.5**” – the exact path can be found during debugging in the debugger pane. In the “Target path” field enter the path where you downloaded and extracted the Qt sources. (The folder which contains qtbase, qtdeclarative, etc. module folders) 

![](/source_file_mapping.png)

### 5.8 Linux

The setup for cross compilation “natively” under Linux is mostly analogous to the Windows instruction. 

At the time of writing this section we used Ubuntu 20.04.

**5.8.1 Install Cross-Compiler Host Packages**

```plaintext
sudo apt install gcc-8-arm-linux-gnueabihf \
g++-8-arm-linux-gnueabihf \
gdb-multiarch \
libglib2.0-dev:armhf
```

**5.8.2 Prepare "sysroot"**

We copy the minimal needed libraries and headers from an existing Docker image:

```plaintext
sudo docker pull mecelectronics/qt-armhf-buster:5.12.5
sudo docker run -it --rm --entrypoint /bin/bash \
-v /:/rootfs mecelectronics/qt-armhf-buster:5.12.5
```

Inside Docker run the following commands to copy them to your host environment (mounted to /rootfs): 

```plaintext
# copy mecqt installation
mkdir -p /rootfs/usr/lib/mec-qt/
cp -r /usr/lib/mec-qt/5.12 /rootfs/usr/lib/mec-qt/
```

```plaintext
# copy OpenGL header files
cp -r /usr/include/EGL /usr/include/GLES /usr/include/GLES2 \
/usr/include/KHR /rootfs/usr/arm-linux-gnueabihf/include/ 
```

```plaintext
# copy OpenGL libraries
mkdir -p /rootfs/usr/local/lib/arm-linux-gnueabihf
cp /usr/local/lib/arm-linux-gnueabihf/libGLES* \
/usr/local/lib/arm-linux-gnueabihf/libEGL* \
/usr/local/lib/arm-linux-gnueabihf/libMali.so \
/rootfs/usr/local/lib/arm-linux-gnueabihf/
```

**5.8.3 Adapt mkspec**

In "/usr/lib/mec-qt/5.12/mkspecs/linux-arm-gnueabihf-g++-mali/qmake.conf" change the compiler variables to match the binaries from the installed Ubuntu package.

Alternatively copy the folder and override the mkspec in QtCreator options for the created Kit.

```plaintext
QMAKE_CC = arm-linux-gnueabihf-gcc-8
QMAKE_CXX = arm-linux-gnueabihf-g++-8
QMAKE_LINK = arm-linux-gnueabihf-g++-8
QMAKE_LINK_SHLIB = arm-linux-gnueabihf-g++-8
```

**5.8.4 QtCreator - Kit Settings**

In “Debuggers” tab add "gdb-multiarch" to be able to debug arm targets.

In “Compilers” tab add cross-compiler binaries

      for C: "arm-linux-gnueabihf-gcc-8"

      for C++: "arm-linux-gnueabihf-g++-8")

In “Qt Versions” tab add mecqt-5.12 copied from docker image (e.g. “/usr/lib/mec-qt/5.12/bin/qmake”) 

In “Kits” tab then combine/select all previously set tools.

**5.8.5 Remote Debugging Using GDB**

GDB can be used for remote debugging from Linux (or Windows WSL). This has the advantage that the debug symbols can be read from the local filesystem rather than transferring them via network for each debugging session. 

The target device needs to run a gdbserver, either manually deployed or installed via the repositories. The host machine (assuming x86\_64 here) must have the package “gdb-multiarch” installed.

Now on the target device start gdbserver on the specified port (here 9999 on all interfaces) with the path to the application to debug: 

`gdbserver :9999 /my/qtapp <args>`

On the host connect to the gdbserver using “gdb-multiarch” command with the same path to the app used for the gdbserver:

`gdb-multiarch /my/qtapp`

Now you can set some properties in the gdb session before starting the debug session.

Set the path to the sysroot of the device from where the libraries should be loaded. 

`(gdb) set sysroot /`

The mecqt packages have their debug symbols extracted (for obvious storage reasons). Each binary just contains a reference path to the extracted file containing the debug symbols. You can tell gdb where to find them:

`(gdb) set debug-file-directory /local/path/to/extracted/dbg/`

If you want to step into the Qt source code you can tell gdb how to map the file reference in the binary to your local filesystem. For example the mecqt packages are compiled from sources in the path “/qtsrc/qteverywhere-src-X.X.X“. Alternatively, you can ensure that the path is the same on your local system.

`(gdb) set substitute-path /qtsrc/qt-everywhere-src-X.X.X /local/path/to/qt-everywhere-src-X.X.X`

Connect to the remote gdbserver:

`(gdb) target remote 192.168.1.2:9999`

For convenience you can put those commands in a file and automatically load and execute them at gdb startup:

`gdb-multiarch -x ~/mecqt.gdb.init /my/qtapp`


### 5.9 Finishing Steps

There are some points which must be kept in mind before making the transition from development to production (or even during development in some cases).

**5.9.1 Default root Password**

The default password for the “root” user is **1** or – on some devices – **dm1devkit**. While this is convenient during development phase it is a major vulnerability when used in a production environment. It is advised to change the root password as soon as the devices could be accessed from the “outside world” to prevent hijacking of the device.

The password can be changed using the passwd command. The password is stored in the **/etc/shadow** file.

**5.9.2 SSH Access**

By default, SSH access via password authentication is enabled. This is a major vulnerability (even more when the root password wasn’t changed) since we already experienced attacks from botnets on the internet scanning for ssh and accessing devices with the default password.

When the device is accessible via the internet the password authentication should be disabled and keyfile authentication should be enabled. Additionally, the default ssh port (22) could be changed to another free port to reduce the risk of an attack.

The ssh server configuration is taken from the file “/etc/ssh/sshd\_config” (see [this](https://linux.die.net/man/5/sshd_config)). For example to disable password authentication set the “PermitRootLogin” option to “without-password” and **append** the public keyfile to the “/root/.ssh/authorized\_keys” file.

After changing the configuration the device needs to be rebooted or the ssh server restarted:

`systemctl restart ssh`

**5.9.3 Read-Write Overlay**{#read-write-overlay}

DM1 Debian Buster devices which have the squalay layer mechanism deployed can enable a special rwoverlay into which all changes in the file system will be written and transparently stacked on top of the data from the squalay images. While this is very useful for development it should be disabled for production.

To en-/disable the rw-overlay edit the fstab file:

```plaintext
mount -o remount,rw /mnt/rootfsimg
nano /mnt/rootfsimg/fstab
```

Make sure one of the following two lines are in the fstab file.

For a read-only filesystem:

`squalay /mnt/root      overlay none 0 0`

For a read-write filesystem:

`squalay /mnt/root      overlay rwoverlay=/mnt/rw/myrwoverlay 0 0`

## 6 GammaRay

[GammaRay](https://www.kdab.com/gammaray) is a tool developed by KDAB to inspect and debug the internals of a Qt application and is very useful to understand how an UI application works.

GammaRay can either be used locally on the host machine or remotely on the target device.

Either way a plugin is needed which matches the used Qt version, since the plugin must be linked against nonpublic Qt API to inspect framework internals.

When running locally the GammaRay host application injects into the running application process. For debugging remotely on the device a so called probe must be deployed which starts the target application and communicates with the GammaRay host application which then visualizes the data from the probe. 

### 6.1 How To

Please see the official user manual: [https://docs.kdab.com/gammaray-manual/latest/](https://docs.kdab.com/gammaray-manual/latest/)

### 6.2 Deploy GammaRay to the Target Device

To deploy the GammaRay device probe (which already comes prebuilt with the MEC toolchain) automatically during the build process add the following line(s) to your qmake project file and (re-)run qmake:

INSTALLS += mec-gammaray

This deploys the GammaRay probe package (to /tmp) on the device. This package must also be extracted so we need to add a custom build step (after the file upload step):

tar xzf /tmp/gammaray-probe-armhf-mecqt5.10.tar.gz -C /tmp

**Note:** check the final filename of the archive to extract in the deployment file list. 

Finally, we need to let GammaRay start the Qt application (so we can connect to it via the GammaRay host application) by overwriting the start command on the target device: 

Alternative start command 

`/tmp/gammaray/bin/gammaray`

with the command line arguments 

`--inject-only <path-to-app> <optional-app-arguments>`

![](/gammaray.png)

## 7 Squalay Diff Application

DM1 Debian Buster devices are using squalay (squashfs layer) images to build up the final filesystem on the device. The images are located in „/mnt/rootfsimg“ and are loaded by alphabetical order, where a higher stacked squalay can overwrite or even hide/delete files of a lower stacked squalay.

There are some base/system squalay images (provided by MEC). Custom squalays (with your adaptions and/or applications) should be stacked on top of the provided squalays.

To assist working with squalay images and give some convenient insight of the contents of a squalay image we created a Desktop application - available for Windows and Linux (Ubuntu 20.04).

### 7.1 Features

• Show final file structure by loading all provided squalay images

• show file attributes (size, permissions, type, …)

• view file contents or save file to local filesystem

• 2-way merge between different versions of the same squalay images

• 3-way merge of squalay images

      especially useful when an update of a base squalay image is provided by MEC and you need to check if it affects your custom layer and adaptions are required to compliy with the update

• show warnings/hints of common mistakes/pitfals in squalay images

### 7.2 Usage

The SqualayDiff application provides inline help directly in the application by pressing F1 and clicking on the corresponding UI element to show an inline help tooltip.

The basic usage is the following:

1.  Load squalay images into A/B/C file list in the top section oft he application window. This can either be done via drag-and-drop or by browsing with the file dialog
2.  Press „Compare“ in the top right corner oft he application to start processing of the provided squalay images and build up the tree
3.  Inspect the tree with one of the 3 filter options:

      a. **Show all files** – No filtering is applied, the whole tree can be inspected.

      b. **Show only changed files** – 2-way diff. The tree now only shows which have changed or were added/removed. Identical files are filtered from the tree view. This is useful to quickly inspect changes between 2 versions of a squalay image. 

      c. **Show only 3-way diff changes** – The tree shows only files which have changed in A/B in comparison to C. This is useful to check layers against an squalay update. For example put the squalays from the base version into A, the updated layers into B and the custom squalays (which where created based on the state in A) into C. All files remaining in the tree now need to be checked and in C updated to the changes made from A to B. 

![](/squalay_usage.png)

## 8 Known Issues / Troubleshooting

### 8.1 Toolchain

**8.1.1 Initial Break upon SIGILL When Debugging**

When debugging remotely on the MEC device using GDB an initially break due to SIGILL signal is expected during loading OpenSSL (libcrypto.so.1.0.0) libs. This is caused by OpenSSL itself to test certain CPU features and is handled by GDB.

Simply continue this break and resume debugging.

This automatic initial break can also be avoided by telling GDB to ignore SIGILL signals at all. This is done by setting “**handle SIGILL nostop**” command to GDB.

To do this in QtCreator go to **_Tools → Options → Debugger_** and open the **“GDB”** tab. Now insert the command in the **“Additional Startup Commands”** field: 

![](/toolchain_debugger.png)

**8.1.2 GDB seems to freeze when stepping into certain shared libraries**

We encountered this strange issue when stepping into a Qt class’ implementation. The GDB command seems to freeze and hang in this state indefinitely.

It needs to be investigated what is causing this issue. Until now there is only a workaround (whenever it seems that GDB doesn’t return a response):

1.  In Debug view (select “Debug” in QtCreator’s left panel)
2.  open the **“Modules”** view via the menu **Window → Views → Modules**
3.  right-click into the Modules view and select **“Update module list”**
4.  after this, GDB should at least leave the frozen state

![](/gdb_freeze.png)

**8.1.3 GDB Shows Incorrect Source File**

It may happen that the debugger stops at a breakpoint inside an incorrect source file. This is highly possible when the application was compiled of multiple files with the same name (e.g. main.cpp). GDB by default sets breakpoints to relative file names. In such cases it’s possible to set breakpoints with absolute file paths instead.

To fix this per breakpoint right click on the breakpoint and select **“Edit breakpoint”**. In the opening dialog select **“Use full path”**:

![](/gdb_shows_incorrect.png)

Alternatively, the same can also be set globally in the QtCreator options (which might not always be desired though):

![](/gdb_shows_2.png)

**8.1.4 Warning “Attribute Qt::AA\_ShareOpenGLContexts Must Be Set before QCoreApplication Is Created.”**

With the mecqt-5.12 toolchain the call to `QtWebEngine::initialize()` throws the following warning in the console:

**Attribute Qt::AA\_ShareOpenGLContexts must be set before QCoreApplication is created.**

This warning is a Qt bug ([QTBUG-76391](https://bugreports.qt.io/browse/QTBUG-76391)) and will be fixed along with the release of Qt 5.14.



### 8.2 QtWebEngine

**8.2.1 Offline Page Render Dead-Lock**

With Qt 5.10 there is a known issue which causes QtWebEngine dead-lock when trying to render it’s “offline” page, means only if QtWebEngine is trying to load a page while there is no internet connection. This issue was only observed when using Qt’s “basic” render loop (by setting the environment variable `QSG_RENDER_LOOP=basic` ).

Even though the default render loop is “threaded”, the “basic” render loop is used to reduce the permanent CPU load and thus also the heat increase of the device.

**Note:** The possible default deployed MecZero UI is affected by this issue since it uses the “basic” render loop by default for the mentioned reasons

## 9 Appendix

### 9.1 Links

**Adjustments / Settings**

|     |
| --- |
| Qt on embedded linux<br><br>[https://doc.qt.io/qt-5/embedded-linux.html](https://doc.qt.io/qt-5/embedded-linux.html) |
| Qt input (touch) adjustments<br><br>[https://doc.qt.io/qt-5/inputs-linux-device.html](https://doc.qt.io/qt-5/inputs-linux-device.html) |

**Development**

|     |
| --- |
| Qt Downloads<br><br>      [http://download.qt.io/](http://download.qt.io/) |
| **QML** |
| Beginners Gettings started tutorial<br><br>      [https://doc.qt.io/qt-5/qtdoc-tutorials-alarms-example.html](https://doc.qt.io/qt-5/qtdoc-tutorials-alarms-example.html) |
| QML Tutorial<br><br>      [https://doc.qt.io/qt-5/qml-tutorial.html](https://doc.qt.io/qt-5/qml-tutorial.html) |
| Code samples<br><br>      [https://doc.qt.io/qt-5/qtquick-codesamples.html](https://doc.qt.io/qt-5/qtquick-codesamples.html) |
| (Unofficial) QML eBook<br><br>      [http://qmlbook.github.io/](http://qmlbook.github.io/) |
| **QtWebEngine** |
| Features<br><br>      [https://doc.qt.io/qt-5/qtwebengine-features.html](https://doc.qt.io/qt-5/qtwebengine-features.html) |
| **QtVirtualKeyboard** |
| Technical Guide (Layouts, Styles, etc.)<br><br>      [https://doc.qt.io/qt-5/technical-guide.html](https://doc.qt.io/qt-5/technical-guide.html) |
| Deployment Guide<br><br>      [https://doc.qt.io/qt-5/qtvirtualkeyboard-deployment-guide.html](https://doc.qt.io/qt-5/qtvirtualkeyboard-deployment-guide.html) |

### 9.2 Environment variables

Here is a collection of useful environment variables:

|     |     |
| --- | --- |
| QT\_QPA\_EGLFS\_FORCE888=1 | Force Qt’s color space to RGB24 (instead of RGB565) - when using EGLFS backend |
| FRONTBUFFER\_LOCKING=1 | fix VSYNC tearing – evaluated by the system OpenGL backend |
| XDG\_CACHE\_HOME | Determines the default cache directory used by the application (used by<br><br>QStandardPaths::writableLocation(QStandardPaths::CacheLocation) method). This cache directory is then for example used by QtWebEngine to store the website cache and by QtLocation to cache the loaded map tiles. |
| XDG\_DATA\_HOME | Determines the default data directory used by the application – This is for example used by QtWebEngine to store cookies etc. to. |
| QML2\_IMPORT\_PATH=/var/myqml | Add an additional import path to look for QML plugins |
| MALI\_NOCLEAR=1 | Prevents the mali display driver to clear the screen when the UI application starts. This prevents a blank screen for a few seconds and enables a fluid transition from the boot logo into the UI application |