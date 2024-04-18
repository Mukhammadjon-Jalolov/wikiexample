---
title: BSPd
description: API to access the hardware from userspace
published: true
date: 2024-04-10T13:40:49.721Z
tags: bspd
editor: markdown
dateCreated: 2024-03-15T13:11:09.509Z
---

# Introduction
The BSPd provides a file based interface for the internal, C-based, BSP interface that defines the API of the hardware abstraction for our devices. Using BSPd, other applications are able to use the BSP abstractions with simple read(2) and write(2) calls. This also enables to control or read the hardware from a shell, via the cat(1) or echo(1) standard POSIX utilities.

> When using shell redirection, BSPd files can only be written by appending to them (e.g. echo 1 >> /tmp/mec/ledr). Overwriting (a single “>”) is not supported because the BSPd doesn’t implement the truncate syscall. However, writing from a C program is done without particular requirements (e.g. fd=open(…, O_WRONLY); write(fd, data, size); ).
{.is-info}


As there may be more than one instance of a particular hardware type, e.g. multiple temperature sensors, the interface usually contains an ID parameter. BSPd contains a mapping from the file name and its read/write permissions to the hardware type and the ID. E.g. the mapping contains something like “{”temp0”, SENSOR_TEMP, 0, O_RDONLY}”, meaning that the file “temp0” refers to the temperature sensor 0 and is read only. Thus, while the basic principle holds (usage by reading and writing files), and a naming convention is used, the file names may differ depending on specific requirements.

It should also be noted, that the files visible in the BSPd mountpoint are virtual. This means, that while the standard file I/O API (open(2), read(2), write(2), close(2)) is used, the content that is returned by read(2) is the value returned from a BSP function (e.g. TEMP_Read), snprintf(3)ed into the buffer passed to read(2) (with maybe a copy inside the Linux kernel). Similarly, content written with write(2) is simply converted as required and passed to the BSP.

# Configuration

## Command 

### General paramters

Currently, the following parameters are supported in the “General” section:
- Mountpoint: Mountpoint for the FUSE subsystem
  Example: Mountpoint=/run/bspd/
  CLI: yes
- Options: Options passed to FUSE (see fuse(8) for the list of supported options)
  Example: Options=-o nonempty
  CLI: yes
  
### Example configuration

The following example configures bspd to mount at /run/bspd, making sure that it mounts even if there are files present in /run/bspd

*bspd -f -o nonempty /run/bspd*


# BSPd interface

After mounting the BSPd, the file list can be retrieved with readdir(3) or ls(1). Depending on the
requirements, multiple files can refer to the same hardware, or a device may not be exported as a BSPd interface.

A limitation of the current BSP implementations is that devices where a value is set (e.g. LEDs or PWM) usually do not provide a matching _Read-function (e.g. there is a PWM_Write() but not a PWM_Read()). Thus the BSPd currently implements read-only or write-only files, but not read/write files.

## Using the interface files

The files can be opened and closed as usual with the open(2)/close(2) call.
If the file is write-only, the returned file descriptor can be written to repeatedly without seeking to offset 0, as the offset is discarded in BSPd. For read-only files, the offset is considered: it is required to call lseek(2) and set the offset to 0 before each read.

The file descriptor can be kept open. In case of reading/writing errors, it is recommended to close and reopen the file, in case BSPd was restarted and the old file descriptor invalidated. Similarly, if the current working directory was changed to the mount point of BSPd, after a restart it is required to change directory with an absolute path, as the current working directory doesn’t exist anymore (so e.g. “cd ..” doesn’t work, but “cd /run/” does).

The content of the files is always a “\0”-terminated C-string, both on read(2) and on write(2). When reading a file, a newline character (“\n”) is included in the content to support line-buffered I/O.
Non-numeric characters are ignored when written to. In case only non-numeric characters (e.g. the word “on”) is written, strtol(3) returns 0 and this value is then passed to the underlying BSP function. E.g. writing “on” to “relais0” would set the value to 0 and actually turn the relay off.

### Example reading code
~~~C
void readTemperature(void) {
	static int fd;
	int tempValue;
	char buf[100];
	if(0 >= fd) {
		fd = open("/run/bspd/legacy/temp0", O_RDONLY))
		if(0 >= fd) {
			perror("open temp0");
			return;
		}
	}

	if(lseek(fd, 0, SEEK_SET) == (off_t)-1) {
		perror("lseek temp0");
		goto badFdError;
	}
	
  if(read(fd, buf, sizeof(buf)) <= 0) {
		//example code - only simple handling of return value
		perror("read temp0");
		goto badFdError;
	}
	tempValue = strtol(buf, NULL, 0);

	//...
	return;
	
  badFdError:
		close(fd)
		fd=-1;

	return;
}
~~~

## Description of interface files

In the recent bspd version the files have been restructured into several groups. All files known from older bspd versions can still be accessed through the folder “legacy”

### Legacy

To allow the user to use the new bspd without changing a lot in his present software we provided the legacy folder, where all files are available, as they were before.

### Common file: Serial number

- Direction: Read-Only
- Type: String
- Unit: none
- File prefix: **serial**
- Returns “*INVALID*” if not available

### Temperature Sensors

- Direction: Read-Only
- Type: Fixed-point decimal with one decimals (e.g. 235 = 23.5°C)
- Unit: °C
- File prefix: **temp** (e.g. temp0, temp1, …)
- Defined sensors:
 	- **temp0**: Internal temperature sensor
  - **ptemp0**: CPU temperature
  - **exttemp0/1**: Temperature based on extadc0/1
- Returns “*INVALID*” if not available

### Temperature Sensors User Offset

- Direction: Read-Only
- Type: Fixed-point decimal with one decimals (e.g.-15 = -1.5°C)
- Unit: °C
- File prefix: **tempx_offset** (e.g. temp0_offset, …)
- Defined sensors:
	- **temp0_offset**: user offset temperature sensor

### Temperature Sensors Compensation Offset

- Direction: Read-Only
- Type: Fixed-point decimal with one decimals (e.g.-15 = -1.5°C)
- Unit: °C
- File prefix: **tempx_compensation** (e.g. temp0_compensation, …)
- Defined sensors:
	- **temp0_compensation**: shows the actual calculated compensation offset for the device

### Humidity Sensors

- Direction: Read-Only
- Type: Fixed-point decimal without decimals (e.g. 23 = 23%)
- Unit: %rH
- File prefix: **humi**
- Defined sensors:
	- **humi0**: Internal humidity sensor
- Returns “*INVALID*” if not available

### Pressure Sensor

- Direction: Read-Only
- Type (Range): Fixed-point decimal with one decimals (10053=1005.3hPa)
- Unit: hPa
- File prefix: **air_pressure**
- Defined sensors:
	- **air_pressure**: internal air pressure sensor
- Returns “*INVALID*” if not available


### eCO2 Sensor

- Direction: Read-Only
- Type (Range): Fixed-point decimal with one decimals (5161=516.1ppm)
- Unit: ppm
- File prefix: **eco2**
- Defined sensors:
	- **eco2**: internal eCO2 sensor
- Returns “*INVALID*” if not available
> Note: after restart of bspd, sensor value is always invalid for 5 minutes, because of internal
> sensor calibration
{.is-info}

### VOC Sensor

- Direction: Read-Only
- Type (Range): Fixed-point decimal with one decimals (5257=525.7ppb)
- Unit: ppb
- File prefix: **voc**
- Defined sensors:
	- **voc**: internal VOC sensor
- Returns “*INVALID*” if not available
> Note: after restart of bspd, sensor value is always invalid for 5 minutes, because of internal
> sensor calibration
{.is-info}

### Air quality index Sensor

- Direction: Read-Only
- Type (Range): unsigned integer
- Unit:
- File prefix: **iaq**
- Defined sensors:
	- **iaq**: internal VOC sensor
- Returns “*INVALID*” if not available
> Note: after restart of bspd, sensor value is always invalid for 5 minutes, because of internal
> sensor calibration
{.is-info}

![aq.png](/bspd_images/aq.png)

### Air quality accuracy

- Direction: Read-Only
- Type: Fixed-point decimal without decimals
- File prefix: **accuracy**
- Defined sensors:
	- Accuracy of internal airquality sensor
- Returns
	- 0… sensor is stabilizing
	- 1… sensor data is too stable to clearly define a reference
	- 2… sensor is calibrating
	- 3… calibration successful


### EXT ADC Sensor

- Direction: Read-Only
- Type (Range): 0...0xFFFFFFFF
- Unit: 32bit adc ticks
- File prefix: **extadc**
- Defined sensors:
	- **extadc0** – backplate input Sens1
	- **extadc1** – backplate input Sens2

### EXT Resistor Sensor

- Direction: Read-Only
- Type (Range): 32bit value
- Unit: Ohms
- File prefix: **extres**
- Defined sensors:
	- **extres0** – backplate input Sens1, based on extadc 0
	- **extres1** – backplate input Sens2, based on extadc 1

### PWM-Actors: 0-10V

- Direction: Write-Only
- Type (Range): unsigned integer (0%(0V)-100%(10V))
- Unit: %
- File prefix: **pwm**
- Defined actors:
	- **extpwm0**: pwm output on the backplate
  
### Relays Actors

- Direction: Write-Only
- Type (Range): unsigned integer (Boolean, 0=off, 1=on)
- Unit: 1
- File prefix: **relais**
- Defined actors:
	- **relais0/1**: Relay 1/2 on the backplate

### Light Sensors

- Direction: Read-Only
- Type (Range): unsigned integer (hardware-dependent)
- Unit: 1
- File prefix: **photo** (e.g. photo0, photo1, …)
- Defined sensors:
	- **photo0**: Internal light sensor

### LEDs

- Direction: Write-Only
- Type (Range): unsigned integer (Boolean, 0=off, 1=on)
- Unit: 1
- File prefix: **led**
- Defined actors:
	- **ledr/ledg/ledb**: Red/Green/Blue LED

### Motion Sensors

- Direction: Read-Only
- Type (Range): unsigned integer (Boolean, 0=no Movement, 1=movement)
- Unit: 1
- File prefix: **motion**
- Defined sensors:
	- **motion0**: Motion sensor based on the internal light sensor

### ToF Sensors

- Config file prefix: tof_config
- Direction: Read/Write
- Data file prefix: tof
- Direction: Read-Only
- Defined sensors:
	- tof / tof_config
- The sensor provides a 16x16 raster on which up to 16 regions can be placed. Each ROI can detect up to 4 objects inside its boundaries.
- Configuration:
	- Configuration via valid JSON (see examples below)
  
		- Available modes are:
▪ ranging : (default) equals single ranging, the ROI is set to the whole FoV
▪ multi-ranging : mode for using multiple ranges, ROI have to be provided

		- Distance modes are:
▪ long : measurements up to 3400mm (default)
▪ medium : measurements up to 1800mm
▪ short : measurements up to 1000mm

		- Polling interval:
▪ Defines how often the sensor is taking measurements
▪ Default: 250ms
▪ Min: 100ms
▪ Max: 10000ms

		- Example ranging: Default config is single ranging in distance mode long, polling interval is
250ms:
	> {
	> &nbsp;&nbsp;&nbsp;&nbsp;"mode":"ranging",
	> &nbsp;&nbsp;&nbsp;&nbsp;"distanceMode":"long",
	> &nbsp;&nbsp;&nbsp;&nbsp;"timeoutms":250,
	> &nbsp;&nbsp;&nbsp;&nbsp;"roi":[[0,15,15,0]]
	> }

	- Example multi-ranging: Multiranging mode, distancemode short, polling interval is 100ms, 16
	ROI (max):
	> {
	> &nbsp;&nbsp;&nbsp;&nbsp;"mode":"multi-ranging",
	> &nbsp;&nbsp;&nbsp;&nbsp;"distanceMode":"short",
	> &nbsp;&nbsp;&nbsp;&nbsp;"timeoutms":100,
	> &nbsp;&nbsp;&nbsp;&nbsp;"roi":[[0, 3, 3, 0], [4, 3, 7, 0], [8, 3, 11, 0], [12, 3, 15, 0],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 7, 3, 4], [4, 7, 7, 4], [8, 7, 11, 4], [12, 7, 15, 4],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 11, 3, 8], [4, 11, 7, 8], [8, 11, 11, 8], [12, 11, 15, 8],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 15, 3, 12],[4, 15, 7, 12], [8, 15, 11, 12], [12, 15, 15, 12]]
	> }
	> 
  
 - Reading the sensor data:
 	- The data read from the tof file is a JSON array, whereas each array entry corresponds to one
	ROI
	- Each array entry can have up to four values corresponding to objects detected in the ROI
	- Example of single ranging result (1 ROI, 2 objects detected):
	>   [[93,2728]]
  - Example of multi-ranging result (16 ROI):
	> [[0],[0],[0],[0],
	> [0],[2708],[2833],[79],
	> [0],[2719],[2768],[80],
	> [0],[0],[0],[0]]
  - Array entries with a value of zero (see example above) are invalid, meaning the sensor was
	measuring out of range (objects too close or too far)
  

## Device

In the “device” path device specific parameters like serial number can be requested.

### serialnumber

- Direction: Read-Only
- Type: String
- Unit: none
- Returns “*INVALID*” if not available

## Display

The “display” path provides all available displays

../display/x/ … x = enumeration value of available displays (e.g. ../display/0/ … local touch display)

### brightness

Set/reads brightness of the backlight

- Direction: Read/write
- Type (Range): unsigned integer (0..100)
- Unit: %

## Sensors

Shows a list of all available sensors of the device

../sensors/x/ … x = enumeration value of available sensor

Only sensors, which are defined for this hardware type are listed in the sensors path.

| **Id** | **Name** | **Description** |
| :-: | :- | :- |
| **0** | CPU | CPU temperature |
| **1** | HDC1080 | Temperature and Humidity Sensor |
| **2** | N/A | |
| **3** | N/A | |
| **4** | ACP20| Power System Management Chip, provides temperature for compensation |
| **5** | BME680 | Gas sensor measuring relative humidity, barometric pressure, ambient temperature, airquality and gas (VOC) |
| **6** | VL53L1X | Time-of-Flight sensor (motion sensor) |
| **7** | SA56004 | Temperature sensor |
| **8** | LM90 External | Temperature sensor |
| **9** | LM90 Internal | Temperature sensor |
| **10** | A111 | Microwave sensor |
| **11** | SC1232AR3 | Microwave sensor |
| **12** | LRADCA20 | |
| **13** | CDM7160 | Deprecated CO2 sensor → EOL |
| **14** | SCD41 | CO2, temperature and humidity sensor |
| **15** | SGP40 | VOC index sensor |
| **16** | SGP41 | VOC index and NOX index sensor |


### Sensor x

All sensors provide a name and status property in the root folder of the sensor. Additionally sensor specific folders are provided according to available sensor features.

#### Name
Provides the name of the sensor, see list above

#### Status
Provides the status of the sensor
0 … sensor ok
Otherwise initialization failed and no sensor specific folders are created

#### Temperature
**value**
compensated temperature of the sensor including user offset
- Direction: Read-Only
- Type: Fixed-point decimal with one decimals (e.g. 235 = 23.5°C)
- Unit: °C
- Returns “*INVALID*” if not available

**rawValue**
actual value of the sensor
- rection: Read-Only
- pe: Fixed-point decimal with one decimals (e.g. 235 = 23.5°C)
- it: °C
- turns “*INVALID*” if not available

**userOffset**
user specific offset added to the compensated sensor value
- Direction: Read/Write
- Type: Fixed-point decimal with one decimals (e.g.-15 = -1.5°C)
- Unit: °C
- Returns “I/O Error” if not available

**.usePT1Filter**
Enable/disable Filter for sensor
- Direction: Read/Write
- Type: unsigned int (0..filter disable, 1..filter enabled)
- Returns “I/O Error” if not available

**.compensationOffset**
Calculated offset to compensate self-heating of the device
- Direction: Read-Only
- Type: Fixed-point decimal with one decimals (e.g.-15 = -1.5°C)
- Unit: °C
- Returns “I/O Error” if not available

#### Humidity
**value**
compensated humidity of the sensor including user offset
- Direction: Read-Only
- Type: Fixed-point decimal with no decimals (e.g. 23 = 23%rH)
- Unit: %rH
- turns “*INVALID*” if not available

**rawValue**
actual value of the sensor
- rection: Read-Only
- pe: Fixed-point decimal with no decimals (e.g. 23 = 23%rH)
- it: %rH
- turns “*INVALID*” if not available

**userOffset**
user specific offset added to the compensated sensor value
- rection: Read/Write
- pe: Fixed-point decimal with no decimals (e.g. 23 = 23%rH)
- Unit: %rH
- Returns “I/O Error” if not available

**.usePT1Filter**
Enable/disable Filter for sensor
- Direction: Read/Write
- Type: unsigned int (0..filter disable, 1..filter enabled)
- Returns “I/O Error” if not available

#### CO2
Actual sensor reading, depending of used sensor could be CO2 or eCO2 value
- Direction: Read -Only
- Type (Range): Fixed-point decimal with one decimals (5161=516.1ppm)
- Unit: ppm
- Returns “*INVALID*” if not available
> 	Note: after restart of bspd, sensor value could be invalid for 5 minutes, because of internal
> 	sensor calibration
{.is-info}

#### VOC
Actual sensor reading
- Direction: Read -Only
- Type (Range): Fixed-point decimal with one decimals (5257=525.7ppb)
- Unit: ppb
- Returns “*INVALID*” if not available
> Note: after restart of bspd, sensor value could be invalid for 5 minutes, because of internal
> sensor calibration
{.is-info}

### VOC Index
Actual sensor reading
- Direction: Read -Only
- Type (Range): unsigned integer, [0 – 500]
- Unit: /
- Returns
	- “*INVALID*” if not available
	- 0 in during startup phase (approx 0.5min)
- Values between 100 - 500 indicates a decline in AQ, values between 0 - 100 indicate an improvement in AQ

#### NOX Index
Actual sensor reading
- Direction: Read -Only
- Type (Range): unsigned integer, [0 - 500]
- Unit: /
- Returns
	- “*INVALID*” if not available
	- 0 in during startup phase (approx 0.5min)
	- 1 indicates a stable NOX concentration, values above 1 indicate an increase in NOX 	concentration

#### Airquality
Actual sensor reading
- Direction: Read -Only
- Type (Range): unsigned integer
- Returns “*INVALID*” if not available
> Note: after restart of bspd, sensor value is always invalid for 5 minutes, because of internal
> sensor calibration
{.is-info}

![aq.png](/bspd_images/aq.png)

**useStaticIAQ**
Enable/disable static IAQ output
- Direction: Read/Write
- Type: unsigned int (0..adaptive IAQ used, 1..static IAQ used)

**accuracy**
Defines the accuracy of a sensor value
- Direction: Read-Only
- Type (Range): Fixed point decimal without decimals
- Values:
	- 0… sensor is stabilizing
	- 1… sensor data is too stable to clearly define a reference
	- 2… sensor is calibrating
	- 3… calibration successful
  
#### Barometer
Actual sensor reading
- Direction: Read -Only
- Type (Range): Fixed-point decimal with one decimals (9957=995,7hPa)
- Unit: hPa
- Returns “*INVALID*” if not available

#### Distance
##### VL53L1xX
**value**
Provides sensor data in json Format
Reading the sensor data:
- The data read from the tof file is a JSON array, whereas each array entry corresponds to one
ROI
- Each array entry can have up to four values corresponding to objects detected in the ROI
- Example of single ranging result (1 ROI, 2 objects detected):
> [[93,2728]]

- Example of multi-ranging result (16 ROI):
> [[0],[0],[0],[0],
> [0],[2708],[2833],[79],
> [0],[2719],[2768],[80],
> [0],[0],[0],[0]]
- Array entries with a value of zero (see example above) are invalid, meaning the sensor was
measuring out of range (objects too close or too far)

**config**
read/write sensor configuration in json Format
- The sensor provides a 16x16 raster on which up to 16 regions can be placed. Each ROI can detect up to 4 objects inside its boundaries.
- Configuration:
	- Configuration via valid JSON (see examples below)
	- Available modes are:
▪ **ranging** : (default) equals single ranging, the ROI is set to the whole FoV
▪ **multi-ranging** : mode for using multiple ranges, ROI have to be provided
	- Distance modes are:
▪ **long** : measurements up to 3400mm (default)
▪ **medium** : measurements up to 1800mm
▪ **short** : measurements up to 1000mm
	- Polling interval:
▪ Defines how often the sensor is taking measurements
▪ Default: 250ms
▪ Min: 100ms
▪ Max: 10000ms
	- Example ranging: Default config is single ranging in distance mode long, polling interval is
    250ms:
	> {
	> &nbsp;&nbsp;&nbsp;&nbsp;"mode":"ranging",
	> &nbsp;&nbsp;&nbsp;&nbsp;"distanceMode":"long",
	> &nbsp;&nbsp;&nbsp;&nbsp;"timeoutms":250,
	> &nbsp;&nbsp;&nbsp;&nbsp;"roi":[[0,15,15,0]]
	> }
- Example multi-ranging: Multiranging mode, distancemode short, polling interval is 100ms, 16
 ROI (max):
	> {
	> &nbsp;&nbsp;&nbsp;&nbsp;"mode":"multi-ranging",
	> &nbsp;&nbsp;&nbsp;&nbsp;"distanceMode":"short",
	> &nbsp;&nbsp;&nbsp;&nbsp;"timeoutms":100,
	> &nbsp;&nbsp;&nbsp;&nbsp;"roi":[[0, 3, 3, 0], [4, 3, 7, 0], [8, 3, 11, 0], [12, 3, 15, 0],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 7, 3, 4], [4, 7, 7, 4], [8, 7, 11, 4], [12, 7, 15, 4],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 11, 3, 8], [4, 11, 7, 8], [8, 11, 11, 8], [12, 11, 15, 8],
	> &nbsp;&nbsp;&nbsp;&nbsp;[0, 15, 3, 12],[4, 15, 7, 12], [8, 15, 11, 12], [12, 15, 15, 12]]
	> }
  

##### SC1232AR3
**value**
Provides sensor data in json Format
In distance detector mode the data read from the value file is a JSON Array containing 5 values in mm, sorted by power/peak_level.

**config**
read/write sensor configuration in json Format

Read or write to this file to get or set sensor parameters. Reading from the file will return the configuration of the momentary running detector.

These parameters are:
| Parameter    | Description                                                                                                                                                |  Type  | Values                                                                                                                                     | Default |
|:------------ |:---------------------------------------------------------------------------------------------------------------------------------------------------------- |:------:|:------------------------------------------------------------------------------------------------------------------------------------------ |:-------:|
| **det**      | which detector to run                                                                                                      | string | md (motion detector)/dd (distance detector)                                                                                                                                      |         |
| **rxgain**   | gain of the internal receiver amplifier                                                                                                                    | string | high/middle/low                                                                                                                            | middle  |
| **interval** | interval between sensing frames in ms                                                                                                                      | number | 8 ≦ interval ≦ 5000 (ms)                                                                                                                   |   20    |
| **ud**       | upper distance of the sensor range (lower_distance < upper_distance ≦ x (cm))                                                                              | number | x=5600 (distance step: normal) / x=2800 (distance step: fine) / x=1400 (distance step: xfine)                                              |   500   |
| **ld**       | lower distance of the sensor range                                                                                                                         | number | 10 ≦ lower_distance < upper_distance (cm) (distance step: NORMAL, FINE)) / 4 ≦ lower_distance < upper_distance (cm) (distance step: XFINE) |   10    |
| **beta**     | configures the coefficient of cumulative average calculation operation to smoothen the signal level obtained from calculation of distance to moving object | number | 0 ≦ beta ≦ 255                                                                                                                             |   205   |
| **ct**       | chirp time, frequency sweep duration in us                                                                                                                 | number | 220/1100/4400 (us) 4400 is unavailable when distance_step is xfine                                                                         |  1100   |
| **dbs**      | digital beam shaper, 95 degree for narrow, 120 degree for wide                                                                                             | string | wide/narrow                                                                                                                                |  wide   |
| **hpf**      | high pass filter                                                                                                                                           | string | fo(first order)/so(second order)                                                                                                           |   fo    |
| **ds**       | distance step, this parameter configures a step of the detected distance of the Distance Detection mode                                                    | string | normal (22.4cm)/fine (11.2cm)/xfine (5.6cm)                                                                                                | normal  |
| **pll**      | peak level lower limit, value configures the lower limit of the peak level corresponding to the distance                                                   | number | 1 ≦ peak level lower ≦ 96                                                                                                                  |   15    |
| **run**      | run or stop a detector                                                                                                                                     | number | 0/1                                                                                                                                        |         |

It is possible to set single values as well as all of them at once via valid JSON. The “det” key is mandatory. While calibrating it is not possible to set any parameters. If a parameter does not fulfill the constraints the write operation will fail.

Some of these parameters need a restart of the detector e.g. *{“det”:”dd”,“run”:0}* followed by a *{“det”:”dd”,“run”:1}*. 

These parameters are:
- rxgain
- chirp time
- digital beam shaper
- distance step
- peak level lower limit

All the other parameters are updated on the fly when writing into config.

Please also note that these values are not saved on the host and need to be set by the application on every start.

#### Presence detection 
##### A111

**value**
Provides sensor data in JSON format. The file is read-only.

The result object contains 3 values:
- presence_detected: this is a Boolean variable, being either 1 on presence detected or 0 if no
	presence has been detected
- presence_distance: if presence has been detected, this property will give the corresponding
	distance to the detected object in mm
- presence_zone: this is a convenience property, indicating the zone in which presence has
	been detected. If no presence has been detected the property will hold the value 0. The value
	corresponds to the n-th slice of ranges calculated as follows:
	- (presence_distance – A111_DEFAULT_STARTING_POINT (== 300mm)) / zone_size
	  (see config section for a description of these properties)

Example:

> {
> &nbsp;&nbsp;&nbsp;&nbsp;"presence_detected":1,
> &nbsp;&nbsp;&nbsp;&nbsp;"presence_distance":3500,
> &nbsp;&nbsp;&nbsp;&nbsp;"presence_zone":19
> }

**config**
read/write sensor configuration in json Format

The config object consists of 4 values (the file is r/w):
- service: at the moment there is only the service “presence_detector” available
- range: two ranges are available
	- short: 0.3m – 1.3 m
	- medium: 0.9m – 4.0m
- timeout_ms: the internal polling interval (20ms – 10000ms)
- zone_size: determines the size of the presence_zone property (1mm – 4000mm)

Example:
> {
> &nbsp;&nbsp;&nbsp;&nbsp;"service":"presence_detector",
> &nbsp;&nbsp;&nbsp;&nbsp;"range":"short",
> &nbsp;&nbsp;&nbsp;&nbsp;"timeout_ms":83,
> &nbsp;&nbsp;&nbsp;&nbsp;"zone_size":250
> }


##### SC1232AR3
**value**
In presence detector mode this read only file will indicate if presence has been detected (1) or not (0).

The value in the file is reset every time the file is read. On device startup, the sensor is set to presence detection by default.

**config**
read/write sensor configuration in json Format

Read or write to this file to get or set sensor parameters. Reading from the file will return the configuration of the momentary running detector.

These parameters are:

| Parameter    | Description                                                                                                                                                                                                                                                                                                        |  Type  | Values                                                                 | Default |
|:------------ |:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |:------:|:---------------------------------------------------------------------- |:-------:|
| **det**      | which detector to run                                                                                                                                                                                                                                                                                              | string | md/dd                                                                  |         |
| **rxgain**   | gain of the internal receiver amplifier                                                                                                                                                                                                                                                                            | string | high/middle/low                                                        |  high   |
| **interval** | interval between sensing frames in ms                                                                                                                                                                                                                                                                              | number | 8 ≦ interval ≦ 5000 (ms)                                               |   20    |
| **ud**       | upper distance of the sensor range                                                                                                                                                                                                                                                                                 | number | lower_distance < upper_distance ≦ 2000 (cm)                            |   500   |
| **ld**       | lower distance of the sensor range                                                                                                                                                                                                                                                                                 | number | 10 ≦ lower_distance < upper_distance (cm)                              |   10    |
| **alpha**    | configures the coefficient of cumulative average calculation operation to<br>smoothen the signal level obtained from calculation of distance to moving object                                                                                                                                                      | number | 0 ≦ alpha ≦ 255                                                        |   230   |
| **ct**       | chirp time, frequency sweep duration in us                                                                                                                                                                                                                                                                         | number | 220 / 1100 / 4400 (us) 4400 is unavailable when distance_step is xfine |  1100   |
| **dbs**      | digital beam shaper                                                                                                                                                                                                                                                                                                | string | wide/narrow - 95 degree for narrow, 120 degree for wide                |  wide   |
| **hpf**      | high pass filter                                                                                                                                                                                                                                                                                                   | string | fo(first order)/so(second order)                                       |   fo    |
| **sc**       | startup count, number of chirps to be discarded before starting the moving object<br>detection operation                                                                                                                                                                                                           | number | 0 ≦ startup_count ≦ 255                                                |   20    |
| **mt**       | motion threshold, configures the threshold to determine if any moving object is<br>detected                                                                                                                                                                                                                        | number | 0 ≦ motion_threshold ≦ 4095                                            |   300   |
| **run**      | run or stop a detector                                                                                                                                                                                                                                                                                             | number | 0/1                                                                    |         |
| **calib**    | this property will start the motion detector calibration process and store the results<br>on the host. The calibration process takes ca 20s during which the read from the config file will return<br>{"calib":1}. There should not be any movement in front of the sensor while doing the calibration<br>process. | number | 0/1                                                                    |         |

It is possible to set single values as well as all of them at once via valid JSON. The “det” key is mandatory.
While calibrating it is not possible to set any parameters.

If a parameter does not fulfill the constraints the write operation will fail.

Some of these parameters need a restart of the detector e.g. *{“det”:”md”,“run”:0}* followed by a *{“det”:”md”,“run”:1}*. These parameters are:
- rxgain
- chirp time
- digital beam shaper

All the other parameters are updated on the fly when writing into config.

> These values are not saved on the host and need to be set by the application on every
> start.
{.is-info}

Calibration:

The calibration can be triggered in every detector mode. Also, it is not necessary to add the “det” key. The whole calibration process takes up to 20s. During the calibration the application does not accept new parameters written into the config file. Reading from it will return {“calib”:1}.

The calibration is stored on the host, so there is no need for the application to run it on every start.

During the calibration itself the sensor scans the environment, therefor it is necessary that there is no motion in front of the sensor for optimal results while calibrating.

> After the calibration the detectors are both stopped, to start them again just set the {“det”: “md/dd”,“run”:1} config.
{.is-info}


## tempcomp

“tempcomp” path provides information about available temperature compensation algorithms available in the device.

../tempcomp/x/… x = enumeration value of available compensation algorithms

Only algorithms, which are defined for this hardware type are listed in the tempcomp path.

| Id | Name | Description |
| :-: | :- | :- |
| **1** | PT1 Compensations | Compensates temperature, using Core Temperature (AXP20) an Reference Sensor (BME680/HD1080) |

### tempcomp x

All compensations provide a name and status property in the root folder of the compensation path.
Additionally compensation specific folders are provided in the parameters sub path.

![tempcomp1.png](/bspd_images/tempcomp1.png)

#### name
Provides the name of the compensation, see list above

#### status
Provides the status of the compensation

0 … compensation ok

Otherwise initialization failed and no compensation specific folders are created

#### PT1 compensation

**description**
Pt1 compensation sums up the result of the individual P control results.

![tempcomp2.png](/bspd_images/tempcomp2.png)
![tempcomp3.png](/bspd_images/tempcomp3.png)
![tempcomp4.png](/bspd_images/tempcomp4.png)

| offset | description |
| :- | :- |
| **base** | Base heating of sensor which should be compensated, without CPU load and Backlight |
| **base_core** | Base heating of device without CPU load and Backlight (AXP sensor is used) |
| **dynam_core** | Dynamic heating of device, dependent on CPU load |
| **dynam_backlight** | Dynamic heating of device, dependent on backlight |

Example:

y~xxx~ = y~xxx_previous~ + ((k~xxx~ * dynamfactor) - y~xxx_previous~) * (interval / (tau~xxx~ + interval))

*base_offset:*
y~base~ = y~base_previous~ + (k~base~ * 1) - y~base_previous~ *(1s / (tau~bas~ + 1s))

**parameters**

| parameter | | description |
| :- | :-: | :- |
| **k_base** | r/w | Proportional factor for base heating. Base heating of sensor which should be compensated, without CPU load and Backlight |
| **tau_base** | r/w  | Time constant for base heating. Base heating of sensor which should be compensated, without CPU load and Backlight |
| **y_base** | r | Calculated offset for base heating. Base heating of sensor which should be compensated, without CPU load and Backlight |
| **k_base_core** | r/w | Proportional factor for base core heating. Base heating of device without CPU load and Backlight (AXP sensor is used) |
| **tau_base_core** | r/w | Time constant for base core heating. Base heating of device without CPU load and Backlight (AXP sensor is used) |
| **y_base_core** | r | Calculated offset for base core heating. Base heating of device without CPU load and Backlight (AXP sensor is used) |
| **k_dynam_core** | r/w | Proportional factor for dynamic core heating. Dynamic heating of device, dependent on CPU load |
| **tau_dynam_core** | r/w | Time constant for dynamic core heating. Dynamic heating of device, dependent on CPU load |
| **y_dynam_core** | r | Calculated offset for dynamic core heating. Dynamic heating of device, dependent on CPU load) |
| **k_dynam_backlight** | r/w | Proportional factor for dynamic backlight heating. Dynamic heating of device, dependent on backlight |
| **tau_dynam_backlight** | r/w | Time constant for dynamic backlight heating. Dynamic heating of device, dependent on backlight |
| **y_dynam_backlight** | r | Calculated offset for dynamic backlight heating. Dynamic heating of device, dependent on backlight |
| **supplytype** | r/w | Powersupply which has been detected according to this device type, default compensation factors are initialized |


| supplytype | description |
| :- | :- |
| **MECONE** | Mecone backplate |
| **USB** | No known backplate detected – we assume device is USB powered |
| **ADC-T40** | Thermostat backplate |
| **ADC-T40-PP** | Powerplate |


## powersupply
In the powersupply path all available power supplies of the device are listed.

../networks/x/ …x = enumeration value of available sensor

| Id | Interface | description |
| :-: | :- | :- |
| **1** | Power supply by mains |
| **2** | Power supply by battery |

![powersupply.png](/bspd_images/powersupply.png)


### powersupply AXP20 AC (Id 1)
This powersupply describes the supply by mains.

| Property | R/W | Description |
| :- | :-: | :- |
| **name** | r | Name of the power supply |
| **status** | r | Initialization status of supply. 0=ok, otherwise error |
| **current** | r | Actual measured current in mA |
| **health** | r | Health of the supply. Valid values: "Unknown", "Good", "Overheat", "Dead", "Over voltage", "Unspecified failure", "Cold", "Watchdog timer expire", "Safety timer expire", "Over current", "Calibration required", "Warm", "Cool", "Hot", "No battery" |
| **online** | r | Indicates present state of the supply. Valid values: 0: Offline 1: Online Fixed - Fixed Voltage Supply 2: Online Programmable - Programmable Voltage Supply |
| **present** | r | Reports whether this supply is present or not in the system. (mainly interesting for batteries) Valid values: 0: Absent 1: Present |
| **type** | r | Describes the main type of the supply. Valid values: "Battery", "UPS", "Mains", "USB", "Wireless" |
| **voltage** | r | Actual supply voltage in mV |


### powersupply AXP20 BAT (Id 2)
This powersupply describes the supply by battery.

| Property | R/W | Description |
| :- | :-: | :- |
| name | r | Name of the power supply |
| status | r | Initialization status of supply. 0=ok, otherwise error |
| batterystatus | r | Represents the charging status of the battery. Valid values: "Unknown", "Charging", "Discharging", "Not charging", "Full" |
| capacity | r | Representation of battery capacity. Valid values: 0 - 100 (percent) |
| current | r | Actual measured current in mA |
| health | r | Health of the supply. Valid values: "Unknown", "Good", "Overheat", "Dead", "Over voltage", "Unspecified failure", "Cold", "Watchdog timer expire", "Safety timer expire", "Over current", "Calibration required", "Warm", "Cool", "Hot", "No battery" |
| online | r | Indicates present state of the supply. Valid values: 0: Offline, 1: Online Fixed - Fixed Voltage Supply, 2: Online Programmable - Programmable Voltage Supply |
| present | r | Reports whether this supply is present or not in the system. (mainly interesting for batteries) Valid values: 0: Absent, 1: Present |
| type | r | Describes the main type of the supply. Valid values: "Battery", "UPS", "Mains", "USB", "Wireless" |
| voltage | r | Actual supply voltage in mV |


## network

Shows a list of all available networks of the device

../networks/x/ …x = enumeration value of available sensor

| Id | Interface name | Description |
| :-: | :- | :- |
| **10** | LTE | LTE network interface |

![network.png](/bspd_images/network.png)

### Network x
All networks provide a generic set of properties in the root folder of the network. Additionally, network specific folders are provided according to network type (e.g. wireless information on wireless network interfaces).

#### name
Provides the name of the network, see list above

#### status
Provides a generic status of the network, comparable to an exit code where 0 means okay and everything else indicates an error state. There is a separate connection-status for networks.

#### connection-status
Provides a text that describes the status of the connection.

Valid values:
"disconnected", "connected"

#### type 
Provides the network type string like “lte”, “wireless” or “ethernet”.

#### ioctl
Writeable property to send specific commands to the network. Takes a json formatted string with command and a data property (if needed, can be omitted otherwise) as input.

e.g.:
> {"cmd":"setConfig","data":{"ip":{"method":1},"lte":{"apn":"apn","pin":1234,"user":"user","pw":"password","auth-method":"CHAP"}}}

When using bash it is important to ensure that the “ stay intact when writing into the ioctl.

**up**
bring up network interface. In case of lte establish a connection to service provider apn.

Bash example: 
~~~bash
echo up >> ioctl
~~~

**down**
take down network interface. In case of lte disconnect connection from service provider apn.

Bash example: 
~~~bash
echo down >> ioctl
~~~

**setConfig**
configure network interface

setconfig messages are provided in json format.
e.g.:
> {
> &nbsp;&nbsp;&nbsp;&nbsp;"cmd":"setConfig",
> &nbsp;&nbsp;&nbsp;&nbsp;"data":
> &nbsp;&nbsp;&nbsp;&nbsp;{
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"ipv4":
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"method":1
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;},
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"lte":
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"apn":"apn",
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"pin":”1234”,
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"user":"user",
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"password":"password",
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"number":"123"
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
> &nbsp;&nbsp;&nbsp;&nbsp;}
> }

#### type specific subfolders

**ipv4**
ip specific properties

only available for network interfaces, supporting ipv4

| Property | R/W | Description |
| :- | :-: | :- |
| **dns** | r | String containing configured dns servers (e.g.: “8.8.8.8 192.168.1.1”) |
| **gateway** | r | Standard Gateway address as string (e.g. “192.168.1.1”) |
| **ip** | r | Ipv4 address of the network interface as string (e.g. “192.168.1.100”) |
| **method** | r | 0=manual (static ip), 1=auto (dhcp), 2=link-local |
| **subnet** | r | Subnet mask of the network interface as string (e.g. “255.255.255.0”) |

**lte**
lte specific properties

only available for network interfaces, supporting lte

| Property | R/W | Description |
| :- | :-: | :- |
| apn | r | Service Provider's Access Point Name |
| number | r | APN profile number |
| pin | r | Pin of sim card |
| pw | r | Password used if any of the authentication protocols are active |
| signal-strength | r | lte signal strength in % |
| user | r | Username used if any of the authentication protocols are active |







