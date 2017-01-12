---
layout: post
title: Emulating Android Magnetic field sensor based on GPS heading
category: Android
---
While working on one of devices, I faced an issue with Google Maps, when it didn't display direction arrow, just position. I already implemented support for GPS, and GPS data contained bearing (heading) value as well, but direction didn't work. After some research I figured out that Google Maps is using compass based on Magnetic Field and Acceleration sensors. Since the device didn't contain these sensors onboard, I should create them programmatically (this post is not about regular Android development, but about customizing Android for some device)...

<!--more-->

# A bit of theory

When you use a compass, its needle rotates towards the "lines" of magnetic field of the Earth pointing to the [Geomagnetic North Pole](https://en.wikipedia.org/wiki/Earth%27s_magnetic_field). Important to note that Geomagnetic North Pole is not the same as Geographic North Pole, which is used by maps and which is the point where all the meridians connect (actually Google Maps uses [Web Mercator Projection](https://en.wikipedia.org/wiki/Web_Mercator), but still). Geographic North Pole is also called as True North.

![https://commons.wikimedia.org/wiki/File:North_Magnetic_Poles.svg#/media/File:North_Magnetic_Poles.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/9/9c/North_Magnetic_Poles.svg/500px-North_Magnetic_Poles.svg.png)

As I said earlier, while experimenting with Google Maps on the phone I figured out that direction pointer is based on programmatic compass. Compass is pretty simple thing and it is using Acceleration and Magnetic Field sensors to calculate Azimuth (you can do it yourself by using [getRotationMatrix()](http://developer.android.com/intl/ru/reference/android/hardware/SensorManager.html#getRotationMatrix(float[], float[], float[], float[])) and [getOrientation()](https://developer.android.com/reference/android/hardware/SensorManager.html#getOrientation(float[], float[]))) methods of SensorManager).

Just to note - your device has three axes which are used in measurements and calculations: X pointing to the right of the device, Y pointing to the top of the device, and Z pointing up to the sky (assuming device is lying on the table). Acceleration and Magnetic Field sensors are using these three axes to present their data.

So, Azimuth is the angle of rotation about device's Z axis or angle between device's Y axis and Geomagnetic North Pole. If Azimuth is zero, you're heading right to the Geomagnetic North. If Azimuth is 90 you're heading East, 180 - South, 270 - West.

![](https://docs.google.com/drawings/d/1girKecjecIb5kfJ50WVRYCKGIXM8ApaHyvgwHqPoxOI/pub?w=328&amp;h=406)

[Acceleration Sensor](https://developer.android.com/reference/android/hardware/Sensor.html#TYPE_ACCELEROMETER) shows acceleration applied to the device among that three axes, including Earth gravity force. When for example your device is lying horizontally on the table, only gravity force is applied to it, and consequently, values would be X=0, Y=0, Z=9.8. Acceleration Sensor is used to determine device's orientation relative to Earth surface.

[Magnetic Field Sensor](https://developer.android.com/reference/android/hardware/Sensor.html#TYPE_MAGNETIC_FIELD) shows the strength of Earth Magnetic Field applied to these three axes, in microteslas. If your device is rotated directly to the Geomagnetic North Pole, Y axis would show max value, X would show zero, and Z would show some value (very very roughly). Exact values may vary significantly depending on your location, so I'll show example with values from 0 to 1, where 1 represents max field strength. So, Magnetic Field Sensor is used to determine device's orientation relative to Geomagnetic North Pole.

![](https://docs.google.com/drawings/d/1cRE1InXjeXWksW3jOUWQ4hWxfFhLQWAu6Qm77S0ri7w/pub?w=726&amp;h=490)

Together these two sensors are used to get Azimuth regardless of device orientation relative to the Earth surface.

### Preparation

As you can see, there is a relation between device rotation around Y axis and Magnetic Field Sensor values, and we can calculate some abstract Magnetic Field values for X and Y axes based on the rotation angle something like this:

```
X = sin(rotation_in_radians) * -1 * BS;
Y = cos(rotation_in_radians) * BS;

Where BS is the Base Strength and could be any arbitrary value, similar to real Earth Magnetic Field strength (say 25 microteslas). It is needed since we don't measure real Magnetic Field but just want to mimic real sensor.
```

As I said, I have a GPS data which contains heading angle. GPS heading is the angle between your moving/driving direction and North provided by GPS receiver in NMEA messages. Where heading of 0 degress means you are heading right to the North, 90 to the East and so on. Exactly what I need!

GPS provides heading for True North and Geomagnetic North as well (for curious ones - I use RMC and VTG messages). And it seems that I should use Geomagnetic North heading since we're going to emulate compass. But after some experiments with Magnetic Field Sensor I figured out that when I provide values corresponging to zero rotation about Z axis, Google Maps shows direction right to the True North, not Geomagnetic North. It seems like it is doing some adjustments, so I decided to use True North heading from GPS.

What about Acceleration Sensor - I'm assuming that my device is lying flat on the table heading its Y axis to the North, and Z axis to the sky, so I just provide static [0, 0, 9.81] values for [X, Y, Z] axes. For the same reason I don't calculate Z axis value for Magnetic Field Sensor.

### Implementation

I'm not going to show you code here, but describe whole concept, thankfully it is not too complicated.

![](https://docs.google.com/drawings/d/164dRNAM2WgVJbK8u7iItNnS1j_QojH49r_AoAWc0y0Q/pub?w=923&h=520)

1. So, GPS data come from GPS receiver to GPS HAL as [NMEA](https://en.wikipedia.org/wiki/NMEA_0183) sentences.
2. GPS HAL is already implemented Android hardware abstraction level for GPS, which parses NMEA messages and provides appropriate data up to Android. It also has Linux server socket, which awaits for client connection and once client is connected, starts writing parsed heading value to that socket.
3. The client is Sensors HAL which connects to server socket and starts reading for heading value. Then it calculates Magnetic Field strength based on heading and provides it as Magnetic Field Sensor value. Acceleration sensor provides just a constant data mentioned above.
4. Google Maps reads Acceleration Sensor and Magnetic Field Sensor values and calculates heading value back.

To add sensors you should implement [Android Sensors HAL](https://source.android.com/devices/sensors/), which is better referenced by [libhardware/sensors.h](https://android.googlesource.com/platform/hardware/libhardware/+/master/include/hardware/sensors.h).

