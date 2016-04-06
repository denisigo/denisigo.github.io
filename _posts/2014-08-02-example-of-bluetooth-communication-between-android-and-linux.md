---
layout: post
title: Example of Bluetooth communication between Android and Linux
category: Android
tags: [Bluetooth, Python]
---

Hi, today I'll show you how to establish Bluetooth communication channel between Android application and Linux desktop using SPP (Serial Port Profile) and virtual serial port. I'm not sure about usefulness of this thing, but I needed it once. As always, I'll include snippets of the code, but full source can be found on [GitHub repo](https://github.com/denisigo/android-linux-bluetooth).

<!--more-->

# Intro to Bluetooth

Bluetooth uses several low-level [protocols](http://en.wikipedia.org/wiki/List_of_Bluetooth_protocols) combined into high-level [profiles](http://en.wikipedia.org/wiki/List_of_Bluetooth_profiles). So we will use [SPP](http://en.wikipedia.org/wiki/List_of_Bluetooth_profiles#Serial_Port_Profile_.28SPP.29) profile which stands for Serial Port Profile which uses [RFCOMM](http://en.wikipedia.org/wiki/List_of_Bluetooth_protocols#Radio_frequency_communication_.28RFCOMM.29) protocol which stands for Radio frequency communication. 

In turn these profiles are implemented by services. Each service has its own UUID (Universally Unique Identifier) to be able to locate desired service during service discovery (using SDP - Service Discovery Protocol). Basic protocols and profiles already have UUIDs which can be found [here](https://www.bluetooth.org/en-us/specification/assigned-numbers/service-discovery). 

SPP profile has **0x1101** for short form UUID and **00001101-0000-1000-8000-00805F9B34FB** for 128-bit form UUID. 

So, when we create a SPP server socket on Android, system assigns unused RFCOMM channel to listen on and registers a service record with defined UUID with local SDP server. Then we discover service records on a target device, find SPP record and get RFCOMM channel number on which our socket is listening. More infromation can be found on [Android Developers portal](http://developer.android.com/guide/topics/connectivity/bluetooth.html).

# Android application

At first we have to declate permissions in our manifest file:

``` xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<!-- We don't need this permission for basic functionality, but
we have to declare it because of android bug, which needs this permission
for BluetoothAdapter.listenUsingRfcommWithServiceRecord.
see http://code.google.com/p/android/issues/detail?id=40608-->
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
```

Then get default Bluetooth adapter. If it returns null, it means device has no support for Bluetooth.

``` java
mBluetoothAdapter = BluetoothAdapter.getDefaultAdapter();
```

Then check if Bluetooth is enabled on device:

``` java
if (!mBluetoothAdapter.isEnabled()) {
  // bluetooth disabled
}
```

You can ask user to enable Bluetooth as shown in Android docs link above. Now, create a server socket using RFCOMM:

``` java
serverSocket = mBluetoothAdapter.listenUsingRfcommWithServiceRecord(
                        SERVICE_NAME, UUID.fromString(SERVICE_UUID));
```

Where SERVICE_NAME is an arbitrary service name, and SERVICE_UUID is SPP UUID: **00001101-0000-1000-8000-00805F9B34FB**. If previous step has passed successfully, we have server socket and are able to listen for client connections:

``` java
bluetoothSocket = serverSocket.accept();
```

Note that this method is blocking and must be invoked from a separate thread. As soon as client is connected, this method will return BluetoothSocket instance. Let's use it to get streams:

``` java
inputStream = bluetoothSocket.getInputStream();
mOutputStream = mSocket.getOutputStream();
```

Now, we can read for messages from input stream:

``` java
byte[] buffer = new byte[1024];

while (!Thread.interrupted()) {
  bytesRead = inputStream.read(buffer);
  byte[] message = new byte[bytesRead];
  System.arraycopy(buffer, 0, message, 0, bytesRead);
}
```

Please note, that we read input stream using a working thread too. And write messages to output stream:

``` java
byte[] data = ...;
mOutputStream.write(data, 0, data.length);
```

That's all the key things in Android app.

# Discovering our service

We will use Python and [PySerial](http://pyserial.sourceforge.net/index.html) library to deal with serial ports. We will use [hcitool](http://manpages.ubuntu.com/manpages/saucy/man1/hcitool.1.html) to discover Bluetooth devices, and [sdptool](http://linux.die.net/man/1/sdptool) to discover some device's services. We will also use [rfcomm tool](http://manpages.ubuntu.com/manpages/precise/man1/rfcomm.1.html) to bind particular device's RFCOMM channel to the virtual serial port. I hope you're able to install these tools by yourself. So, enable Bluetooth on your device and enable device visibility for discovering. Now, let's discover devices from desktop using hsitool:

``` shell
$ hcitool scan

Scanning ...
	98:DD:D0:EA:62:4E	Lenovo A369i
```

Great, we have discovered our device and it has following MAC: 98:DD:D0:EA:62:4E. Now, start our Android application and let's discover Bluetooth services of our device using its MAC:

``` shell 
sdptool records 98:DD:D0:EA:62:4E

....

Service Name: TestService
Service RecHandle: 0x10013
Service Class ID List:
  UUID 128: 00001101-0000-1000-8000-00805f9b34fb
Protocol Descriptor List:
  "L2CAP" (0x0100)
  "RFCOMM" (0x0003)
    Channel: 21
```

Great, we have discovered our SPP service, take a look at Protocol Descriptor List - that's protocols our service is using, and there is RFCOMM one on channel 21. So, it's time to create our virtual serial port:

``` shell
sudo rfcomm bind /dev/rfcomm0 98:DD:D0:EA:62:4E 21
```

Where /dev/rfcomm0 is the name of our virtual serial port device, then go MAC address and port number. Now, it's time to create our Python client!

# Python client

Create and open serial port:

``` python
port = serial.Serial(port='/dev/rfcomm0', baudrate=921600, timeout=1)
port.open()
```

Create and start reading thread:

``` python
def readThread(port):
  while True:
    # Read some bytes available from port
    bytes = port.read(1024)
    # Decode them as a string
    message = bytes.decode("utf-8")
    if message:
      print message

thread.start_new_thread( readThread, ( port, ) )
```

And then take all from input and write to the port:

``` python
while True:
  inp = raw_input('>')
  if len(inp):
    port.write(inp)
```

See full client source [here](https://github.com/denisigo/android-linux-bluetooth/blob/master/BluetoothClient.py). The end.
