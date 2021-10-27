# arsdk_documentation
This is a rewriting of the communication protocol used by Parrot drones in a format that is easier for me to follow, and with fixed external links.

It is heavily based on the protocol documentation that can be found [here](https://developer.parrot.com/docs/bebop/ARSDK_Protocols.pdf).

# Basic Information

Parrot products communicate over one of two possible networks: Wi-Fi or BLE. At this time, only the Wi-Fi protocol is documented below as it is the one used by the model of drone that I have for testing (Parrot ANAFI). However, the differences, with the exception of device discovery, are minimal.

All data sent on the network is in little endian byte order.

# Discovery

## Wi-Fi

Wi-Fi products advertise themselves on their network using multicast DNS (mDNS). The service type is of the format `_arsdk-CODE._udp.local.`, where CODE is a two-byte hex code identifying the product. These codes can be found [here](https://github.com/Parrot-Developers/libARDiscovery/blob/master/Sources/ARDISCOVERY_Discovery.c), but some are unhelpfully named. The ANAFI identifies itself as `_arsdk-0914._udp.local`, and the code `0x0914` is very helpfully referred to by libARDIscovery as 'Unknownproduct_5'.

From mDNS you will be able to access the following information:

Info | From | Description
-----|------|------------
Name | Service name | Name of the product (this is typically the same as the name of its wifi network)
IP | Service address | IP address of the product.
Port | Service port | Connection port for the TCP handshake (see below)
Serial number | txtData | The serial number of the product. txtData is a JSON string, device_id within it is the serial number.

# Connection

## Wi-Fi

A TCP handshake needs to be performed with the product using the IP and port obtained from the mDNS service advertisement. To accomplish this, establish a TCP connection and send a JSON string consisting of the below information: (note: seems like there is some extra information here for other things that [ARSDK_Protocols.pdf](https://developer.parrot.com/docs/bebop/ARSDK_Protocols.pdf) does not document, so this may be due to be updated)

Key | Value
----|------
d2c_port | The UDP port you want the product to send you data on. (d2c = device to controller)
controller_type | A string describing your controller (phone, tablet, computer, ...)
controller_name | A string naming your controller (application/program name)
device_id | The serial number of the product you are connecting to. Optional, but if it is provided and does not match the product's serial the connection will be refused, so it is a useful safety feature.

The device will respond with another JSON string of the following format:

Key | Value
----|------
status | If non-zero, connection was refused. The error codes can be found [here](https://github.com/Parrot-Developers/libARDiscovery/blob/master/Includes/libARDiscovery/ARDISCOVERY_Error.h).
c2d_port | The UDP port you should send data to. (c2d = controller to device)
c2d_update_port | The FTP port for sending product updates.
c2d_user_port | Another FTP port ([ARSDK_Protocol.pdf](https://developer.parrot.com/docs/bebop/ARSDK_Protocols.pdf) does not explain further, but I am assuming it is for e.g. downloading video files from the SD card) Optional.
arstream_fragment_size | Size of ARStream fragments. Optional.
arstream_fragment_maximum_number | Maximum number of ARStream fragments per video frame. Optional.
arstream_max_ack_interval | Max time between ARStream ACKs. Optional.
skycontroller_version | The SkyController version. Only present on SkyControllers, obviously.

### Disconnection Detection

Devices will assume the remote to be disconnected if no message has been received in 5 seconds.

# Communication

## Wi-Fi


### Overview and Terminological Confusion

The protocol documentation uses some terminology that I disagree with, especially in the section about its UDP protocol where the terms `packet` and `frame` are used backwards to their normal meaning. However, I am going to use the same naming convention for the sake of anyone trying to read that and this at the same time.

With that in mind, the full block of data in the UDP datagram is referred to as a `packet`. Each `packet` can contain one or more `frame`s, which are composed of a `header` and `data`.

### Frame Structure

The header of a frame consists of the following information, which will be expanded on below:

Name | Size | Description
-----|------|------------
Data type | 1 byte | The type of frame this is (see below)
Buffer ID | 1 byte | The buffer in which this frame should be stored.
Sequence number | 1 byte | The sequence number of this frame within its buffer .
Total size | 4 bytes | The size of the entire frame, header included.
Data | Arbitrary | Data. The size of this section will be total size - 7 bytes, since the header is 7 bytes.

#### Data Type

There are four types of data.

Name | Value | Description
-----|------|------------
ACK | 1 | This frame is an acknowledgement of data sent previously.
DATA | 2 | This frame contains data, and does not need to be acknowledged.
LOW LATENCY DATA | 3 | The same as data, but should internally be given a higher priority.
DATA WITH ACK | 4 | This frame contains data, and must be acknowledged (via a returned ACK frame)

### Buffers

#### A note on necessity...

Personally, I'm not sure if this concept of buffers really needs to be replicated, at least writing in a higher level language, and it is sufficient to just use data types and buffer IDs to route frames to their proper locations on the product being controlled. That being said, I will document buffers regardless for the sake of completion.

#### Definition

Buffers are FIFO instances with specific parameters affecting their behavior. There are 256 buffer IDs (0-255), but they are unidirectional so in practice there are up to 512 buffers as for each ID there is a send and a receive buffer. Buffers have a data type and an ID, and these are intended to be the data type and ID sent in the frame header (i.e., commands are placed into the outgoing buffer of type DATA and ID 32, and when they are popped they are sent on the network with these values.) The following parameters define a buffer in the ARSDK:

Name | Description
-----|------------
ID | ID of the buffer from 0 to 255 inclusive.
dataType | Type, as will be sent in the frame header.
ackTimeoutMs | Time before considering a frame lost (only relevant for buffers of dataType DATA WITH ACK)
numberOfRetry | Number of retries before considering a frame lost (same note as above)
numberOfCell | Size of the FIFO.
dataCopyMaxSize | Max size of an element in the FIFO.
isOverwriting | For adding to sending buffers: whether to drop old data, or refuse new data. For adding to receiving buffers: the same, but note that refused DATA WITH ACK is not acknowledged, so that it will be retried.

#### ID Ranges

Buffers 0 to 9 are reserved for internal usage. Buffers 10 to 127 are used for sending data (i.e. DATA, LOW LATENCY DATA, DATA WITH ACK). Buffers 128 to 255 are used for acknowledgement (i.e. ACK) - the ID to acknowledge data on buffer x is x + 128, e.g. if you receive DATA WITH ACK on buffer 11, you send ACK on buffer 139.

By convention, data buffers for c2d data grow upwards from 10, and data buffers for d2c data grow donwwards from 127.

#### Reserved Buffers

Buffers 0 to 9 are reserved, but only buffers 0 and 1 are used at this time. They implement a ping protocol which works as follows:

1. Send a [timespec](https://en.cppreference.com/w/c/chrono/timespec) of the current time on buffer 0.
2. The receiver immediately sends back the same timespec on buffer 1.
3. The difference between the time in the timespec and the time it was received back is the latency.

This is not mandatory to implement, and the product will default to assuming a latency of 1 second if the pong is not implemented.

#### Existing Buffers

The below details existing buffers and the settings of their parameters. Note that different devices have different buffers, so this section may not be complete. Please note that 'used for' is only a suggestion, as devices will essentially accept any command in any buffer.

##### Acknowledgement Buffers

All buffers from 128 to 255 (i.e. the ACK buffers) have the same settings:

Key | Value
----|------
ID | dataBufferID + 128 (as described above)
dataType | ACK
ackTimeoutMs | 0 (irrelevant)
numberOfRetry | 0 (irrelevant)
numberOfCell | 1
dataCopyMaxSize | 1
isOverwriting | 0

##### Bebop, SkyController, ANAFI, ...

###### c2d 

* Used for piloting/camera control commands

Key | Value
----|------
ID | 10
dataType | DATA
ackTimeoutMs | 0 (irrelevant)
numberOfRetry | 0 (irrelevant)
numberOfCell | 2
dataCopyMaxSize | 128
isOverwriting | 1

* Used for events, settings, ...

Key | Value
----|------
ID | 11
dataType | DATA WITH ACK
ackTimeoutMs | 150
numberOfRetry | 5
numberOfCell | 20
dataCopyMaxSize | 128
isOverwriting | 0

* Used for emergency commands

Key | Value
----|------
ID | 12
dataType | DATA WITH ACK
ackTimeoutMs | 150
numberOfRetry | -1 (infinite)
numberOfCell | 1
dataCopyMaxSize | 128
isOverwriting | 0

* ARStream video acks (not 254 :( )

Key | Value
----|------
ID | 13
dataType | LOW LATENCY DATA
ackTimeoutMs | 0 (irrelevant)
numberOfRetry | 0 (irrelevant)
numberOfCell | 1000
dataCopyMaxSize | 18
isOverwriting | 1

###### d2c

* Periodic sensor reports from the device

Key | Value
----|------
ID | 127
dataType | DATA
ackTimeoutMs | 0 (irrelevant)
numberOfRetry | 0 (irrelevant)
numberOfCell | 20
dataCopyMaxSize | 128
isOverwriting | 1

* Used for events, settings, ...

Key | Value
----|------
ID | 126
dataType | DATA WITH ACK
ackTimeoutMs | 150
numberOfRetry | 5
numberOfCell | 256
dataCopyMaxSize | 128
isOverwriting | 0

* ARStream video data

Key | Value
----|------
ID | 125
dataType | LOW LATENCY DATA
ackTimeoutMs | 0 (irrelevant)
numberOfRetry | 0 (irrelevant)
numberOfCell | arstream_fragment_maximum_number\*2 (from TCP handshake)
dataCopyMaxSize | arstream_fragment_size (from TCP handshake)
isOverwriting | 1

### Sequence Numbers and Acknowledgement

Each buffer has its own independent sequence number. This number is incremented for each new frame sent, but not for retries. Products will ignore out of order or duplicated data using these sequence numbers, but will still ACK them if relevant. If the gap in sequence number between stored and received is too high (ARNetwork uses 10), then rather than considering the frame out of order it will set its stored sequence number to the one received.

Acknowledgement is accomplished by sending an ACK frame to the relevant buffer ID (i.e. the buffer ID of the frame you are acknowledging + 128), where the data part of the frame is the sequence number of the frame being acknowledged.

The frame:

Key | Value
----|------
Data type |  DATA WITH ACK
Buffer ID |  11
Sequence number | 34
Total size | 12
Data | (whatever)

is acknowledged by the frame:

Key | Value
----|------
Data type |  ACK
Buffer ID |  139
Sequence number | 1 (ack buffer has its own sequence number!)
Total size | 8
Data | 34

# Commands

## Introduction

Commands are the actual data sent in data frames. They represent a control command to the device, or sensor updates/settings/warnings from the device. All commands are documented in several large XML files which can be found [here](https://github.com/Parrot-Developers/arsdk-xml/tree/master/xml).

Individual commands are grouped into classes, which are grouped into projects or features. For humans, a command can be identified by `ProjectName.ClassName.CommandName`, for instance `ARDrone3.Piloting.TakeOff`. For sending on the network, commands are identified by a triplet:

 Key | Size
----|------
Project/Feature ID | 1 byte
Class ID | 1 byte
Command ID | 2 bytes

For example, in the linked GitHub repository, within ardrone3.xml, you can find:

```xml
<project name="ardrone3" id="1">
	All ARDrone3-only commands
	<class name="Piloting" id="0">
		All commands related to piloting the drone
		<cmd name="TakeOff" id="1">
			<comment
				title="Take off"
				desc="Ask the drone to take off.\n
				On the fixed wings (such as Disco): not used except to cancel a land."
				support="0901;090c;090e;0914;0919"
				result="On the quadcopters: the drone takes off if its [FlyingState](#1-4-1) was landed.\n
				On the fixed wings, the landing process is aborted if the [FlyingState](#1-4-1) was landing.\n
				Then, event [FlyingState](#1-4-1) is triggered."/>
			<expectations>
				<immediate>
					#1-4-1(state: motor_ramping)
					#1-4-1(state: takingoff)
				</immediate>
			</expectations>
		</cmd>
  </class>
</project>
```

Thus, the command which we can refer to as `ARDrone3.Piloting.TakeOff` is identified on the network by the tuple `(1, 0, 1)`.


## Arguments

Commands can have a variety of arguments, which are also specified and described in the XML. For instance, `ARDrone3.Piloting.PCMD` (which is too large for me to paste into here directly) has `<arg name="flag" type="u8">`, `<arg name="roll" type="i8">` and so on.

Arguments can have the following types:

Type | Size | Description
-----|------|------------
u8/i8 | 1 | unsigned/signed 8 bit integer
u16/i16 | 2 | unsigned/signed 16 bit integer
u32/i32 | 4 | unsigned/signed 32 bit integer
u64/i64 | 8 | unsigned/signed 64 bit integer
float | 4 | IEEE-754 single precision
double | 8 | IEEE-754 double precision
string | ? | Null-terminated string
enum | 4 | enum as defined in xml

As an example of enums, `ARDrone3.PilotingState.FlyingStateChanged` has this argument:

```xml
<arg name="state" type="enum">
		Drone flying state
		<enum name="landed">
			Landed state
		</enum>
		<enum name="takingoff">
			Taking off state
		</enum>
		<enum name="hovering">
			Hovering / Circling (for fixed wings) state
		</enum>
				...
</arg>
```

If the drone reports that it is in the landed state, then the value of this argument will be 0. Hovering is 3, et cetera.

## Packing into Frame

A command is placed into the data section of a frame by placing the identifying tuple first, and then the arguments in order.

## Some extra attributes in the XML

The XML contains some extra attributes which serve as hints about how a command should be used.

### buffer

This can be `NON_ACK`, `ACK`, or `HIGH_PRIO`, with the default if unspecified being `ACK`. It hints as to what buffer the command should be sent to.

### timeout

This can be `POP`, `RETRY` or `FLUSH`, with the default if unspecified being `POP`. It hints as to what to do if an acknowledged frame was unable to be sent after the maximum time/retries. 

* `POP`: Just remove this frame from the buffer.
* `RETRY`: Keep retrying (infinitely)
* `FLUSH`: Empty the entire buffer.

### listtype

I can't find a single command that actually has this attribute, so I can't redocument it in good faith. Check the relevant section of [ARSDK_Protocols.pdf](https://developer.parrot.com/docs/bebop/ARSDK_Protocols.pdf).


