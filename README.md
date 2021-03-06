## Linux AppleMIDI & CMIDID Kernel Module

CMIDID (Configurable MIDI Driver) is an experimental Linux kernel driver which 
can be used to build custom lowcost MIDI keyboards with breadboards, jumpwires
and buttons.

This repositoy includes another kernel driver which implements the RTP MIDI
(aka AppleMIDI) realtime protocol.

It's possible to use several Linux machines with CMIDID drivers to send
MIDI events over the network and combine the events via AppleMIDI into a
single MIDI output.

###DISCLAIMER

We are not responsible for any damage to your system. Neither software nor hardware.

Use at your own responsibility 

## Preparing your Raspberry 

In order to build kernel modules on your Raspberry Pi, running the current version of raspbian, it is necessary to install the kernel sources as they are not shipped in a convenient debian package.

To facilitate the process of determining the according commit hash of your build a little script exist.

Just follow the tutorial at https://github.com/notro/rpi-source/wiki

## AppleMIDI

The applemidi driver is able to create a bridge between an alsa sequencer port and any applemidi compatible network client.

It is ported from a project called midikit (https://code.google.com/p/midikit/) from Jonas Pommerening under BSD-3 license.

The logical location within the kernel can be described as follows:

Alsa sequencer -> Applemidi -> Network Stack -> any compatible client

The Protocol is based on RTPMIDI specified in ~~RFC 4695~~ RFC 6295 (http://tools.ietf.org/html/rfc6295) with a proprietary extension from Apple.

The extension includes the otherwise separately performed session management functions directly into the protocol and is simple to implement. This allows a compatibility of a huge group of implementations:
*	CoreMIDI (Apple)
*	nmj (Java Android)
*	Arduino AppleMidi
*	rtpMIDI (Windows)
*	...

### Installation
The installation of the kernel module is straight forward.

After cloning the repository

	git clone https://github.com/jtee/cmidid-kerneldriver.git .

to your raspberry pi, you have to build the module.

	cd applemidi
	make
	
The build process should run without any warnings.

If this was successful you have the binary module `applertp.ko`

#### 64 bit divmod

This module is intended to be used on a raspberry pi. As there exists no 64 bit division or multiplication on the ARM, this has to be emulated in a software trap.
This is included by the `_uldivmod.S` assembly code.

If you want to use this module on an architecture where the `__aeabi_uldivmod` symbol is available you have to remove the `_uldivmod.o` target from `applertp-objs` in the Makefile.

### Usage

To use the module you have to load it with the desired port for the control channel. This is required to be done as root

	sudo insmod applertp.ko port=5008

__Remember__: The control port and the next higher have to be free, otherwise the module will fail to listen on these ports.

If you omit the port parameter, it will listen on 5008 and 5009 by default.

You should use an even port number to avoid any incompatibilities.

After loading the module it will register a new alsa sequencer card and a client with capabilities of accepting midi events.

You can verify this by listing the possible output ports of alsa with

	aconnect -o
	 
where there should be one entry like

	Client ##: 'AppleMidi' [Typ=Kernel]
    	0 'AppleMidi Port-0'

Now you can route any midi input channel to this by

	aconnect source:port applemidi:0

replacing source:port with the client number and port (most probably 0) and applemidi with the client number of the AppleMidi output.

### Clients

Your system is now also listening on two udp ports.

On any supporting other system you can now connect to this service by its hostname or IP and the port you have assigned.

After connecting, the midi events will be received over the network and can be further processed.

If you connect multiple clients to one server, all of them will receive the midi events.


### Unload

If you are ready and no longer need the service you can shut it down with

	sudo rmmod applertp
	
Therefore you must not have any aconnect clients connected. You can flush all connections easily by

	aconnect -x

### Remarks

####Wifi

__DON'T USE WIFI__
because the CDMA/CA algorithm will not deliver the network packets as deterministic as a fully switched ethernet, where ideally no collisions happen.

To test this, you can run the small program in

	cmidid-kerneldriver / playground / userspace_midi / equi.c
 
compiled with `gcc -std=gnu99 -lasound -o equi equi.c` (you may need the libasound2-dev package) which sends equidistant notes. Connect this in aconnect to the applemidi driver and listen over Wifi. Then you can hear how equidistant these notes are received.

#### IPv6

The server is not capable of handling IPv6 addresses.

#### Journal

_This feature is not yet implemented._

As UDP does not acknowledge Packets, there can be undetected loss. The option to use TCP instead was not chosen because of the overall overhead, which leads to delays.
To compensate for this, the rtpMIDI standard specifies a journal which logs all sent events for every client. The AppleMIDI session management then periodically sends feedback packets to acknowledge the reception of packets up to some sequence number. All earlier journal logs can then be discarded.

### Internals

#### Initialisation
Up on initialization the following call-trace is taken.

mod_init
* `MIDIDriverAppleMIDICreate`
    * allocate main driver structure
    * `MIDIDriverInit`, initialize alsa connection
        * `ALSARegisterClient`, create alsa card and client
        * `MIDIClockProvide`, init a clock for timestamping of events
    * `_applemidi_connect`, initialize networking
        * `sock_create`, create control socket
        * `sock_create`, create rtp socket
        * `sk->sk_data_ready`, set receive callback function
    * `RTPSessionCreate`, create RTP session
        * select source id (`ssrc`)
    * `RTPMIDISessionCreate`, create  RTPMIDI session
    * `setup_timer`, start up periodical timer for synchronisation

#### Removal
On unload all allocated structures are freed and peer connections are hung up.

mod_exit
* `MIDIDriverAppleMIDIDestroy`
    * `_applemidi_disconnect`, hang up clients
        * `_applemidi_disconnect_peer`, end peer sessions and free data structures
        * `sock_release`, release network sockets
	* free sessions
	* `del_timer`
	* free driver structure
	
#### Callbacks

When fully loaded, the module reacts on three types of callbacks.

##### Timer timeout

The timer callback `_applemidi_idle_timeout` is called every 5 seconds and cycles through all peers. With each it initiates a synchronization handshake.

##### Network receive

When a UDP packet is received on one of the listening sockets, `_socket_callback` is called to handle the packet.

If first checks, if it is a valid AppleMIDI packet (`_test_applemidi`) and then tries to process the command (`_applemidi_recv_command`).
There it dissects the packet byte-wise and fills the `command` structure of the driver.

_As only receiving session invitations is implemented, session initiation is ignored._

After successful parsing the driver composes the according resonse (`_applemidi_respond`).

Invitations are all accepted and session termination packets are processed with deleting the peer.

For synchronization packets the timestamps are calculated and responded (`_applemidi_sync`).

All outgoing commands are handled by `_applemidi_send_command` which assembles binary packets from `command` structures and sends them over the specified socket.

##### Alsa event

Alsa events from connected producers are catched by the `_alsa_input` callback. This repackages a `snd_seq_event` to a `MIDIMessage` object and passes this to `RTPMIDISessionSend`.

This creates the appropriate RTPMIDI packet for each peer:
* per peer journal
* differential timestamps

The content is then passed to `RTPSessionSendPacket` for encapsulation in a RTP packet.

There information as the ssrc and the rtp payload type is prepended and finally sent from the rtp socket.


## CMIDID Kernel Module

### Building the Module

It's recommended to use the provided Makefile `module/Makefile` to build the CMIDID kernel module in the same directory.

### Dependencies

Check the required module dependencies with
```
$ modinfo cmidid.ko
```
after building the CMIDI module and load them accordingly. Currently only the modules `snd` and `snd-seq` are required.

### Module Parameters

The CMIDID module requires some parameters to specify the properties of the custom MIDI device.

* `gpio_mapping`: Maps from an integer array of GPIO port numbers and note
values to a single MIDI keyboard key. The array should contain several triples
of values, where the first two integers in each triple are the GPIO port
numbers which will be used as input buttons for a keyboard key and the third
value is the note associated with this key.
So, e.g. `gpio_mapping=17,4,60` assigns the GPIO ports 17 and 4 as input 
buttons for a key. Port 17 is the start button and port 4 the end button, so
that the key is considered touched if there's an interrupt event on port 17
and the key is pressed if there's a subsequent event on port 4. In this example,
the MIDI note value 60 (= C4) is set for this key.
__NOTE__: The size of this array must be a multiple of three and the maximum
number of MIDI keys is defined in `cmidid_main.h` with `#define MAX_KEYS`.

* `jitter_res_time`: Assuming that hardware buttons are connected to the GPIOs,
there's the possiblity to use software resolution of button jittering/bouncing.
This parameter specifies the time (in nanoseconds) after an interrupt event on
a GPIO port, during which subsequent interrupts are ignored.
According to the tests we did, setting `jitter_res_time=1000000` (= 1 ms) is a
reasonable choice. Note that this will delay the note-on and note-off events
sent after a key press by the same amount of time.

* `start_button_active_high` and `end_button_active_high`: Those values should
be set to 0 or 1, depending on whether the hardware input buttons are connected
to pull-up or pull-down circuits. Assigning `start_button_active_high=1` will
consider the first button (of each custom keyboard key) pressed, if there's
a rising edge event on the corresponding GPIO port.
__NOTE__: Values other than 0 and 1 will result in undefined behaviour.

* `stroke_time_min` and `stroke_time_max`: These parameters are used to set
the key hit time thresholds (in nanoseconds) which are used to determine
the maximum and minimum key hit velocity. If the time delay between hitting
the start and the end button for a single key is less than or equal to
`stroke_time_min`, then the velocity of the note-on event will be 127 (which
is the maximum MIDI velocity). If the time delay is greater than of equal to
the `stroke_time_max`, the event velocity will be zero.
IOCTl can be used to specify the interpolation function, which calculates the
velocity values inbetween.

* midi_channel`: An integer value from 0 to 15 which sets the MIDI channel for
the CMIDID MIDI device. This can be used by MIDI synthesizers which receive
MIDI events on mutliple channels to assign a unique instrument to each channel.

### IOCTL Configuration

The kernel module creates a device  `/dev/cmidid` which is only used for ioctl
communication. The user space programm `module/ioctl_test` has a commandline
interface to communicate with the module. The source `module/ioctl_test.c` can
be used as a reference for the available ioctl commands.

Ioctl can be used to set various interpolation functions for the MIDI event
velocities and it can be used for transposing, i.e. adding a constant
(positive or negative) value to each sent MIDI note.

The availabe command values are defined in `cmidid_ioctl.h`.

### Using the Local Audio Port

It is possible to synthesize the midi stream generated by the kernel module
directly on the Raspberry Pi. This can be done for example with the open source
software synthesizer fluidsynth, which is available in the offical repo of
raspbian. Also a soundfile it necessary which can be easily found on the web.
A sample configuration can be found in the `module/prepare_module.sh` script,
which starts fluidsynth in the background. After that to module must be loaded 
and get connected to fluidsynth. This can be done automatically by the 
`module/rewire_module.sh`

## Building an Instrument

### Kernel Modules

If you sacrifice a raspberry for your fancy new instrument, you probably do
not want to set everything up after each boot.
It is possible to put a compiled version of the kernel modules in the
```
/lib/modules/`uname -r`/build/
```
directory so it can be found by `modprobe` and be loaded at boot time.
The configuration for this depends on you distribution (most probably 
`/etc/modules` or `/etc/modprobe.d`).

### Hardware

For building an actual hardware instrument it is necessary to dive a tiny
little bit into electronics. This is also means, that you can __destroy__ real
hardware (raspberry pi) if you screw things up.

The buttons are best attached by a pull-up circuit:

![Pull up circuit](/documentation/circuits/pull-up.png)

Be careful! Use the 3.3V pin and not the 5V, otherwise you would break your
GPIO.


### Example Setup

An example setup could look like this:

![Pull up circuit](/documentation/setup.jpg)

Two raspberry pis were used. The one on the left with 2 keys (4 buttons) and
the right one with 4 keys (8 buttons).

These are connected via 100 MBit ethernet.

To synthesize the audio it is preferable to use more powerful hardware as the
MacBook Pro shown with Logic. In direct comparisons of the path
```
GPIO -> cmidid -> alsa -> fluidsynth
```
versus
```
GPIO -> cmidid -> alsa -> applemidi -> network -> Logic
```
the latter one had less delay. This evolves from the high performance
requirements of the synthesizer, which are not reached by the Raspberry Pi.

