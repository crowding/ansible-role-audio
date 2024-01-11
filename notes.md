# Linux Audio setup / Troubleshooting

## NOTE

The following describes a manual setup/troubleshooting process; I
actually implement my audio setup via Ansible; so any time this talks
about adding packages or writing things to config files, that's
implemented in the Ansible role. This file details some of the
troubleshooting I did on the way to a having that config.

## Goals

I want to be able to stream music from any mobile device (laptop,
iDevice, Android, guest's phones) to any room in the house. I also
want to be able to take the content I'm listening to upstairs and
move it downstairs, or to other rooms as I move around.

I also want to be able to connect my Bluetooth headphones to it, and
have audio routed there automatically, and be able to walk around the
property (so a long range transmitter, or streaming on my phone)

This should also support me wandering all the way around with my
headphones. (perhaps I should be able to connect my phone as a client
to the stream?)

I might wish there to be a web management console to connect the streams.

Now, there are already tutorials for multiroom audio on the Raspi, but
they require you to have a Raspi for each endpoint. My house's layout
is such that you can reach almost every room from a location near the
furnace, so it makes a lot of sense to have one central server with
several analog outputs.

This is also because the DACs that come on these devboards are all kind
of crap.

For these instructions I am running on Armbian Wheezy install.

## Hardware Setup

* one [Orange Pi Plus 2E][orangepi] running armbian. This is based on
  the Allwinner H3 chip.
  * I had originally tried this with a Raspberry Pi, but ran into
    problems with its USB implementation. The Rpi had two USB2.0
    ports, but on closer inspection these two ports were fed by a hub
    on one bus, and worse, it was a passive hub.  A passive hub is a
    problem because talking to a USB 1.0 device, like many DACs, the
    entire bus slows down to USB 1.0 speed. But wait, there's more!
    The Ethernet adapter sits on the USB bus, which further constrains
    bandwidth. And newer Raspberry Pi devices have continued to put
    the ethernet adapter on USB.
* [One 7-port USB 2.0 hub][dlink]. This was initially required because
  of the Raspi's USB shenanigans (it actually has only one USB
  tranceiver connected to a PASSIVE hub! So communicating with a
  USB1.1 device on one port eats all the time slices that could be
  used communicating with faster devices. An active hub
  will do rate conversion which helps with this.)
* A few USB audio DACs. I have a few Griffin iMics, which have the
  interesting property that they clock themselves off the USB port
  rather than an internal clock -- this means you can run more than
  one of them without them getting out of sync, without resampling.
* I also picked up a 5.1 channel USB DAC, with the idea that maybe it can be
  usable as two or three synchronized stereo outputs.
* An [HDMI DAC][splitter] might also also a possibility to carry up to
  three or four stereo signals. If the kernel for the H3 supported
  audio over HDMI, which [at the time of writing it does not][kernel].

[orangepi]: http://www.orangepi.org/orangepiplus2e/
[dlink]: http://www.amazon.com/gp/product/B00008VFAF/ref=oh_details_o00_s00_i01?ie=UTF8&psc=1
[splitter]: https://www.amazon.com/Monoprice-BlackbirdTM-HDMI-Audio-Extractor/dp/B01B5FR722/ref=sr_1_2?dchild=1&keywords=7.1+hdmi+dac&qid=1596154189&sr=8-2
[kernel]: https://linux-sunxi.org/Linux_mainlining_effort#Status_Matrix

### Changing the video out resolution on the Orange Pi Plus 2E

I would like to use a 1600x1200 panel with this thing (or another of
these ARM boxes.) It looks like my armbian install has:

    Armbian Linux 3.4.113-sun8i

Might I want to upgrade this to a mainline 4.x kernel? Or, wait,
they're on 5.x now. It's been a while since I touched this project.

Started over with a fresh Armbian install.

This install seems to be using my native resolutions just fine (on the
console, haven't tried with Xwindows yet.)

## Software setup.

This time I'll be using Ansible to make this setup reproducible.

I have Windows, Mac and Linux devices on my home network, so the
solution should have connectivity to all.  Here's a short list of
network sound protocols:

* Airplay is native on Mac and iOS devices
* PulseAudio is standard on Linux aimed at general desktop usage
* JACK, another Linux option more aimed at music production setups
* Miracast? DLNA? What does Android use for network audio?
* Bluetooth

* The [`shairport`][shairport] package emulates an AirPlay audio
  endpoint, which can be driven from iOS devices.
* Audio server protocols like JACK, PulseAudio, Airplay and Miracast
* Bluetooth

[shairport]: https://github.com/abrasive/shairport

I want to support enough methods to span my many sources, reaching
any or multiple endpoints, and without contention (i.e. if multiple
devices want to stream to the same speakers, mix them together.)

## Getting ALSA working

First, I need to get ALSA working. (Note, [this page][erj],
interestingly, seems to reccommend using OSS v4. I might come back to
investigate that when I have a low latency application.)

[erj]: http://insanecoding.blogspot.com/2009/06/state-of-sound-in-linux-not-so-sorry.html

### List ALSA playback devices:

Provided your devices are supported, you can ask ALSA to list them:

```
peter@speakers:~$ aplay --list-devices
**** List of PLAYBACK Hardware Devices ****
card 0: Device [USB Sound Device], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: system [iMic USB audio system], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: ATR2USB [ATR2USB], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 3: DG60 [Avantree DG60], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 4: Codec [H3 Audio Codec], device 0: CDC PCM Codec-0 [CDC PCM Codec-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 5: allwinnerhdmi [allwinner-hdmi], device 0: 1c22800.i2s-i2s-hifi i2s-hifi-0 [1c22800.i2s-i2s-hifi i2s-hifi-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

This shows some different USB soundcards I have connected, plus the
onboard audio DAC and the HDMI output.

`aplay -L` will also list several PCMs per card, like `surround51`. I
think these are generated by ALSA corresponding to the formats
supported by the card.

```
peter@speakers:~$ aplay -L
...
surround51:CARD=surround,DEV=0
    USB Sound Device, USB Audio
    5.1 Surround output to Front, Center, Rear and Subwoofer speakers
```

### Assign stable device names with udev rules

ALSA assigned names like "card 0", "card 1" to the cards it supports.
The problem with this is that USB devices, being hot-pluggable, will
generally not get assigned to the same numbers between restarts. So
you need some way to make the assignment between device and card name
stable.

One way to do that is [writing udev rules][udev] that
assign each device a name based on its vendor ID or path.

[udev]: http://www.alsa-project.org/main/index.php/Changing_card_IDs_with_udev

To get the hardware path of each device, you can run `udavadm monitor`
and then plug/unplug each device. This will show you the sysfs paths
for each device as you plug and unplug it. This will be something
like,
`/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.4/4-1.4:1.0/sound/card4`,
where the 'cardN' is dynamically assigned. 

Using this information, you can now write a file named something like
`/etc/udev/rules.d/39-audio.rules` which would look something like
this (here with one 5.1-channel and one 7.1-channel USB adapters that both look
identical to udev)

```
SUBSYSTEM!="sound", GOTO="sound_alsa_end"
ACTION!="add", GOTO="sound_alsa_end"
DEVPATH=="*/4-1.3:1.0/sound/card*", ATTR{id}="surround5"
DEVPATH=="*/4-1.5.3:1.0/sound/card*", ATTR{id}="surround7"
LABEL="sound_alsa_end"

SUBSYSTEM!="sound", GOTO="sound_pulse_end"
ACTION!="change", GOTO="sound_pulse_end"
KERNEL!="card*", GOTO="sound_pulse_end"
DEVPATH=="*/4-1.3:1.0/sound/card*", ENV{PULSE_NAME}="surround5"
DEVPATH=="*/4-1.5.3:1.0/sound/card*", ENV{PULSE_NAME}="surround7"
LABEL="sound_pulse_end"
```

Note that this matches `add` events for `ATTR{id}` but matches a
`change` event for `ENV{PULSE_NAME}`. In principle I could have
discovered this by carefully watching udevadm monitor, but I
discovered it in this [message][].

[message]: https://lists.freedesktop.org/archives/pulseaudio-discuss/2016-November/027206.html

The udev rules can also match on the vendor and brand of the device,
which might be more convenient, if you don't have devices with identical
chipsets.

I actually generate this file from Ansible via a [template][],
generated from the corresponding [hostvars file][] where I collect
information about the setup in one place.

[template]: templates/audio_udev_rules.j2
[hostvars file]: example_setup/host_vars/speakers.local.yaml

After writing the rules file, apply the rules using:

    udevadm control --reload-rules && udevadm trigger

then reconnect your devices. Then you should be able to see the names
in the output of `aplay -l` as well as in the directory
`/proc/asound/` and `pacmd list-cards`.

## Configuring virtual devices in asound.conf

We may also want to configure some virtual devices via ALSA. One
example of a virtual device might be to output just the rear or center/sub
channel of the 5.1 decoder. Another example of a virtual device might
be one virtual device that outputs to multiple physical devices.

### Splitting a surround sound device into virtual 2-channel devices

With a 7.1 surround card, you can supply four rooms with simultaneous
stereo tracks, without having to worry about clock skew.

#### Getting information from `/proc`

To find out the values for altset, you can poke around in the /proc
filesystem. For example, `/proca/asound/.../stream0` contains a
description of the different formats that the sound device supports.

```
peter@speakers:~$ cat /proc/asound/surround5/stream0
USB Sound Device at usb-1c1c000.usb-1.3,
full speed : USB Audio
 
Playback:
  Status: Stop
  Interface 1
    Altset 1
    Format: S16_LE
    Channels: 8
    Endpoint: 6 OUT (ADAPTIVE)
    Rates: 44100, 48000
    Bits: 16
```

An example `asound.conf` for this might look like this: (see also [this page][splitting]):

```
pcm.surround5 {
   type dmix
   ipc_key 2048
   slave { pcm { type hw; card surround5; rate 48000 }; channels 6 }
}
 
pcm.lab {
  type plug
  slave {
    pcm {
      type plug
      slave { pcm surround5 }
      ttable.0.0 1
      ttable.1.1 1
    }
    channels 2
  }
  route_policy default
}
 
ctl.lab { type hw; card surround5 }
 
pcm.laundry {
  type plug
  slave {
    pcm {
      type plug
      slave { pcm surround5 }
      ttable.0.2 1
      ttable.1.3 1
    }
    channels 2
  }
  route_policy default
}
 
ctl.laundry { type hw; card surround5 }
```

[splitting]: https://alsa.opensrc.org/Splitting_front_and_rear_outputs_.asoundrc

Writing the asound.conf gets quite repetitive so I plan to
template it out from Ansible.

## Testing ALSA playback

Playing from the command line to a specific device. Here we're playing to a virtual device named "lab"

```
mpg123 -o alsa -a lab avril_14th.mp3
```

### <a name="multiple"></a>Playing simultaneously to multiple outputs using ALSA (but see below)

ALSA has the ability to gang virtual devices together, but I won't be
using it.  The reason is because of clock skew -- generally, each
sound card will have its own clock, and two clocks will slowly drift
apart over time.  There are some exceptions, wlike the chips used in
the Griffin iMic, which sync their clocks to the USB bus. If you have
synched sound outputs, here's how you'd set up virtual multiple output
targets:

```
 # -- output the same stereo signal to two outputs each of both
 # -- surround 5 and surround 7 cards
pcm.gang1 {
  type plug;
  slave.pcm {
    type multi;
    slaves.a { pcm surround5; channels 6 }
    slaves.b { pcm surround7; channels 8 }
    bindings.0 {slave a; channel 0};
    bindings.1 {slave a; channel 1};
    bindings.2 {slave a; channel 4};
    bindings.3 {slave a; channel 5};
    bindings.4 {slave b; channel 2};
    bindings.5 {slave b; channel 3};
    bindings.6 {slave b; channel 6};
    bindings.7 {slave b; channel 7};
  }
  route_policy duplicate; # duplicates stereo across channels
}
```

Note that this can also be done using our existing named PCM
definitions as:

```
pcm.gang2 {
  type plug
  slave.pcm {
    slaves.a { pcm { type plug; slave { pcm front5; rate 48000 }; }; channels 2 }
    slaves.b { pcm { type plug; slave { pcm front7; rate 48000 }; }; channels 2 }
    bindings.0 {slave a; channel 0};
    bindings.1 {slave a; channel 1};
    bindings.2 {slave b; channel 0};
    bindings.3 {slave b; channel 1};
  }
  route_policy duplicate; # duplicates stereo across channels
}
```

But this form seemed to cause a substantial start-up delay, and probably
more CPU overhead.

### Relaying to Pulse

Instead, we relay the output from libalsa to PulseAudio, (which will
send its data back to kernel ALSA drivers, but that's neither here nor
there.) Do this using the alsa pulse plugin, for example:

    pcm.everywhere { type pulse; device "everywhere" }
    ctl.everywhere { type pulse; device "everywhere" }

# Installing Pulse

Last time the above is what I did to make multiple outputs. This time
I want to see if PulseAudio can help. Install pulse with ```apt-get
install pulseaudio-module-zeroconf``` (which will also install the main
pulseaudio service.)

In previous attempts with a Raspberry Pi on raspbian, I failed to get
pulse PulseAudio. It would try to talk to ALSA and then ALSA would
talk back to pulseAudio, or something.... I maybe wasn't entirely
clear on the Linux sound architecture.

But based on [this blog entry][blog] (by one of the authors of PulseAudio), I
think that it matches my use case better than JACK. So I might try again.
[blog]: http://0pointer.de/blog/projects/when-pa-and-when-not.html

List your audio devices with `pacmd list-cards`.

This told me that all the cards I'd named and configured in ALSA
configs didn't carry over -- they have some complicated USB path name
that that gained from scanning `/proc` Noe, when I do a `pacmd list-cards`
it sees both the ALSA hardware cards, but it doesn't see the virtual
plugs. I thin I will recapitulate those in pulseaudio land anyway. But
more worrisome is, it doesn't see the card names that I set in udev
rules?

### Directing Pulse to use ALSA-lib to detect cards

It looks like by default, pulseaudio interprets the contents of udev
by its own rules to make sound cards. This is controlled in
`/etc/pulse/default.pa` I commented out the lines about automatic
hardware detection and added:

    load-module module-alsa-sink device=surround7 tsched=0
    load-module module-alsa-sink device=surround5 tsched=0

(doing this without `tsched=0`, [led to a lot of chops and
dropouts][dropouts].)

[dropouts]: https://wiki.archlinux.org/index.php/PulseAudio/Troubleshooting#Choppy_sound_with_analog_surround_sound_setup

When changing pulse configuration, restart Pulse with the commands
`pulseaudio -k; pulseaudio -v -D`.

### Directing Pulse to talk to cards on its own

We can also tell Pulse to talk to kernel ALSA directly. In `default.pa`:

    load-module module-udev-detect

might do the trick. If not, we can connect cards manually:

    load-module module-alsa-card device_id=surround5 card_name=alsa_card.surround5 rate=48000 sink_name=surround5 tsched=0

If we applied the udev rules above, this should result in seeing the card names set by those rules:

    pacmd list-cards | grep "\(index: \|name: \)"
        index: 0
            name: <alsa_card.platform-hdmi-sound>
        index: 1
            name: <alsa_card.surround5>
        index: 2
            name: <alsa_card.surround7>
        index: 3
            name: <alsa_card.platform-1c22c00.codec>

You may be asking, "hold on, what's the difference between
`module-alsa-sink` and `module-alsa-card`? Moreover, how is this "not
using ALSA?"

The terminology is confusing because there are two things called ALSA:
the kernel modules that talk to sound cards, and the userspace library
`libalsa` (which is what is configured using `asound.conf`.) When
using `module-alsa-sink` pulse talks to `libalsa` which may in turn
talk to the kernel; when using `module-alsa-card` Pulse talks to the
kernel directly.

## Testing Pulse outputs

To play to a Pulse output:

    mpg123 -o pulse -a alsa_card.surround7 avril_14th.mp3

## Splitting outputs

To split a surround sound device into virtual stereo devices, I wrote
lines like the following in `default.pa`

    load-module module-remap-sink sink_name=laundry sink_properties="device.description='laundry'" remix=no master=alsa_output.surround5 channels=2 master_channel_map=front-left,front-right channel_map=rear-left,rear-right

This makes a virtual output named "laundry" that takes a stereo signal
and redirects it to the

Test with:

    mpg123 -o pulse -a laundry avril_14th.mp3

### My outputs are getting redirected to the default!

I was having problems with several outputs reing redirected to the
first output. I traced this to the line placed at the end of default.pa:

    set-default-sink lab

which appeared to _redirect_ the sinks that were defined earlier in
the file. I'm not sure what the reason for this behavior is. The
evidence was lines like this in the output of `pulseaudio -v`:

    I: [pulseaudio] sink.c: The sink input 3 "(null)" is moving to lab due to change of the default sink.

Placing the `set-default-sink` ahead of the definitions didn't work; the
solution was to place the `set-default-sink` after the _first_ sink was
defined. I also disabled the line:

    load-module module-default-device-restore

### ALSA and Pulse channels don't map the same.

Note that this uses channel "names" rather than channel outputs. In
Pulse the surround channels come in a stereotyped order. Are they the
same order as ALSA exposes? Given both alsa and pulse names for the
outputs, I should check if they map the same. They do not.

ALSA is mapping like this: (what ALSA channel numbers - what device outputs)
* 0/1 - front
* 2/3 - center/lfe
* 4/5 - rear
* 6/7 - side

Whereas Pulse is mapping (what claimed in default-pa - what device outputs):
* "front-left,front-right" - front-left,front-right
* "front-center,lfe" - rear
* "rear-left,rear-right" - center/lfe
* "side-left,side-right" - side-left,side-right

So the rear and center/LFE pairs are swapped.

For the present purposes I fixed the mapping in the template file used to generate `default.pa`.

## Combining different outputs into one virtual sink.

Now you want to play a single source over multiple sinks. This can be
setup in one line like this:

    load-module module-combine-sink sink-name="LR_K_O" slaves="kitchen,office,living_room" channels=2

Once I spelled everything right, this actually does start to work a
bit better than the ALSA way.

## Running Pulse as a service

I would like Pulse to start at system load. By default, it is started
by the graphical desktop on login. I want to have it started at system
load and function for multiple users.

First, in `/etc/pulse/client.conf`, place

    autospawn = no
    default-server: "unix:/tmp/pulse-server"
    enable-memfd: "yes"

To make the Pulse server accept connections over Unix sockets,in `/etc/pulse/default.pa`

    load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1;10.0.0.0/24;192.168.0.0/24 auth-anonymous=1
    load-module module-native-protocol-unix auth-group=audio socket=/tmp/pulse-server
    load-module module-zeroconf-publish

Then I made a file in `/etc/systemd/system/pulseaudio.service` to
start pulseaudio as a daemon:

    [Unit]
    Description=Pulse Audio
    Documentation=man:pulseaudio(1)
    After=sound.target

    [Service]
    Type=simple
    ExecStart=/usr/bin/pulseaudio -v
    User=pulse
    Group=audio

### system mode is unnecessary?

Now, PulseAudio has a setting called [system mode][], which requires
you to start the server as root in multi-user audio setups. And there
is [a page about the dire consequences of doing so.][bad idea]. The
thing is, I was able to make shairport-sync connect, from one user
account, to a pulseaudio daemon running under a different user
account, without running in system mode. And I know it's not running
in system mode because it's obeying the configuration in `default.pa`
and not that in `system.pa`. So it seems like system mode isn't even
necessary, in the headless server scenario. `systemd` takes care of
daemonizing and insulating pulseaudio in its own user account.

Now, there is already a "user" pulseaudio service defined in
`/usr/lib/systemd/user/pulseaudio.service` and
`/usr/lib/systemd/user/pulseaudio.socket`. This apparently is how the
logged-in user requests a pulseaudio server to start up -- systemd
spawns the server on connection to the socket. I wound up deleting the
pulseaudio files in `/usr/lib/systemd/user/` (as well as the symlinks
that point to them) and creating similar files in
`/etc/systemd/system` to define a system service. Again, I found it
was _not_ necessary to use pulseaudio's "system mode;" systemd is
perfectly capable of starting a program under a daemon account.

[system mode]: https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/SystemWide/
[bad idea]: https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/

### Fixing <a name="permissions">permissions</a> with ALSA backend

Running off of systemd, `shairport-sync` (and other services we might
add) runs under its own siloed user account.  With the pulse daemon
running under my main account, I have this when I try to play from
shairport-sync:

    Sep 18 03:21:31 localhost shairport-sync[912]: ALSA lib pcm_dmix.c:1052:(snd_pcm_dmix_open) unable to create IPC semaphore
    Sep 18 03:21:31 localhost shairport-sync[912]: alsa: error -13 ("Permission denied") opening alsa device "bedroom".

This is coming from the ALSA level -- `shairport-sync` is loading
ALSA, ALSA is parsing `asound.conf`, finding the [dmix plugin][] in
front of the sound card, and attempting to connect to inter-process
port associated with it, but the running copy of Pulse has already
grabbed the sound card and isn't letting this random daemon talk to
it. The fix is to add a couple of `ipc` options to `asound.conf`:
[dmix plugin]: https://alsa.opensrc.org/Dmix

    pcm.surround7 {
       type dmic
       ipc_key 2048
       ipc_perm 0666
       ipc_key_add_uid false
       ....
    }

This makes it so that any user on the local machine can connect.

That does it for the individual sinks (which currently use ALSA in a
way that skips Pulse.) However, when I play to the multiple sinks
(which forward from ALSA to pulse) I am getting this:

    Sep 18 05:55:40 localhost shairport-sync[935]: alsa: error -111 ("Connection refused") opening alsa device "everywhere".
    Sep 18 05:55:40 localhost shairport-sync[935]: ALSA lib pulse.c:242:(pulse_connect) PulseAudio: Unable to connect: Connection refused

This error is coming from the Pulse client level (here running from
the pulse plugin loaded from userspace ALSA.)

Now, the pulse server did not output anything. So the client lib must
just be failing to open the pipe?

Actually, on restarting the pulse server, I am getting this in the logs:

    Sep 18 07:56:22 localhost pulseaudio[29068]: Loaded "module-native-protocol-tcp" (index: #16; argument: "auth-ip-acl=127.0.0.1;10.0.0.0/24 auth-anonymous=1").
    Sep 18 07:56:22 localhost pulseaudio[29068]: Failed to parse module arguments
    Sep 18 07:56:22 localhost pulseaudio[29068]: Failed to load module "module-native-protocol-unix" (argument: "auth_group=audio socket=/tmp/pulse-server"): initialization failed.

This was due to misspelled options -- wherever I copy-pasted from used
a dash, not a hyphen.

### Pulse is crashing/rebooting over and over

When I connect to Airplay, I see Pulse rebooting itself over and over,
until eventually it starts playing. This happens when I'm using mpg123
from the command line, too, so it's not airplay or networking that's
the problem.

When running pulse as user rather than systemd, running mpg123, it
doesn't do this?

Next suspect is realtime. What if I disable realtime in pulse daemon,
still running from systemd? Nope.

Logs show:

    pulseaudio[4791]: We are idle, quitting...

Okay, so I boneheadedly set `exit-idle-time = 0` thinking that would
disable idle exit (which I saw no need for bc headless server) and
instead that made it reboot itself several times while it was supposed
to be negotiating a connection.

Nope, still get many reboots running from systemd and using
mpg123. How can I monitor what systemd is doing?

### Pulseaudio running from systemd: Pacmd: "No PulseAudio daemon running"

Running `pacmd list-sinks` fails with `No PulseAudio daemon running, or not running as session daemon.`

I'm getting this error when running `pacmd` (either as root or as a
user in the `audio` group.). It's nonsense because I am successfully
playing audio through Pulse!

It turns out that this is a permissions issue? Because when I `sudo -u
pulse pacmd list-cards` it works. But when do a plain "sudo" it
doesn't! A search turns up this [hint][]:

> One disadvantage of this arrangement is the commands pacmd and pactl will no longer work when run as your user. Both rely on the user dbus session instead of the socket for communicating with the pulseaudio daemon.
[hint]: https://gist.github.com/Earnestly/4acc782087c0a9d9db58

So it seems I'll just have to use `sudo` for this (and probably run
the web UI under the same user?)

### Enable verbose logging

To enable verbose logging, run `sudo -u pulse pacmd set-log-level 4`

To follow logs use:

    sudo journalctl -f | grep '\(pulseaudio\|shairport\)'

## Sinking system audio from Windows?

## Exposing a web UI?

## Auto switch on and off amplifiers?

A class D amp like the famous Lepai can be left on constantly while
wasting little energy.  Some of my other amps have a bigger quiescent
power draw, though, so it would be nice to power them on and off
automatically.

I have a Home Assistant install on this same machine, and a Zigbee
controller running from

## Acting as a Bluetooth sink

After AirPlay, Bluetooth is the next most important protocol to
support. (IDK what Windows uses for an over-the-network audio
protocol?).

Since my current system doesn't have a bluetooth radio, I plugged in a
USB one. Finding an adapter that works with Linux can be tricky. I am
using a BT4.0 adapter based on a CSR chipset, which seems well
supported. But as far as BT 5.0 goes, ??? There is recently support in
the kernel for Realtek RTL8763B chip, but I can find no non-fake USB
dongles using that chip yet.

Assuming Bluetooth is running and Pulse knows about it, we can try
pairing it to my phone [using bluetoothctl][]. I followed Ubuntu's
[pairing][] instructions. For authentication, bluetoothctl asked me
to check that the same six-digit code was displayed on both ends, and
I was able to play from my phone and the audio came out of
pulseaudio's default sink. 
[using bluetoothctl]:
https://computingforgeeks.com/connect-to-bluetooth-device-from-linux-terminal/
[pairing]: https://core.docs.ubuntu.com/en/stacks/bluetooth/bluez/docs/reference/pairing/inbound

But then the connection broke, and would not reconnect.

### Always discoverable

Permanently turn on discoverability by editing `/etc/bluetooth/main.conf`. and setting...

### [TODO] Multiple virtual devices?

I would like to be able to drive multiple bluetooth devices, or select
the output that the bluetooth goes to. Can I make my dongle appear as
multiple bluetooth devices? There is a mention in
`/etc/bluetooth/main.conf` of something called "Multiple Profiles Multiple Devices"

    # Enables Multi Profile Specification support. This allows to specify if
    # system supports only Multiple Profiles Single Device (MPSD) configuration
    # or both Multiple Profiles Single Device (MPSD) and Multiple Profiles Multiple
    # Devices (MPMD) configurations.
    # Possible values: "off", "single", "multiple"
    #MultiProfile = off

The Bluetooth spec has this to say:

> *Single Profile Multiple Devices (SPMD):* In this configuration, a
> single profile is used concurrently between several Bluetooth
> devices. For example, one device runs multiple instances of the same
> profile and each instance is connected to a separate Bluetooth device
> supporting that profile.

This sounds like what I want -- to run multiple A2DP sinks that connect to different devices.

Then, I think I need to load the bluetooth module multiple times in pulseaudio?

[TODO]

### PIN authentication

I would like to authenticate with a PIN, so that I don't have to ssh
in and run `bluetoothctl` to pair things. This behavior is controlled
by what blueZ calls an "[agent][]," which is some external program that runs and is 
[agent]: https://www.kynetics.com/docs/2018/pairing_agents_bluez/

    [bluetooth]# agent off
    Agent unregistered
    [bluetooth]# agent KeyboardOnly
    Agent registered
    [bluetooth]# default-agent
    Default agent request successful

But when I do this, it doesn't pair. It connects then immediately disconnects.

Can I make this more verbose? meanwhile, what does blueman do?

The last post in this [thread][] sheds some light.
[thread]: https://www.linuxquestions.org/questions/linux-wireless-networking-41/setting-up-bluez-with-a-passkey-pin-to-be-used-as-headset-for-iphone-816003/



### [TODO] pacmd won't connect?



### Attaching Bluetooth speakers


## [TODO] Web UI for bluetooth and Pulse control

Since this is a headless server, there are some things that still
require manual intervention such as Bluetooth pairing and switching
Pulse outputs.

## [TODO] Virtual sink to the garage?

In the garage, which isn't physically wired to the rest of the house,
I have an Apple Airport Express -- an ancient one that
[modern MacOS refuses to configure][express], even. It has an airplay
output. Can I add it to this system? What I want is for PulseAudio to
discover the Airport device and add it to the available sinks.

[express][]
[raop][https://www.reddit.com/r/linuxaudio/comments/cvlg5d/configuring_airplay_speaker_as_pulseaudio_sink/]

## [TODO] Dynamically adding Blutooth sinks?

I would also like to be able to dynamically connect Buetooth devices
(outdoor speakers).

I also want to be able to listen with my phone over wi-fi.

There should be a virtual sink called "wireless" and when a Bluetooth
sink or streaming client connects, it will receive anything sent to
"wireless" sink.

There ought be be some UI for connecting bluetooth devices, exposed on
the web.

Another way to do this might be to use a Bluetooth adapter that is a
small sound card of its own, which I happen to have laying
around. It's a bit of a bain to connect this one up though.

## Web control panel?

Now the nice thing would be if there could be a web interface that
shows the matrix of what is playing to where. Alas, mostly we just
have web interfaces to a volume control.

https://github.com/Siot/PaWebControl

# Supporting network audio protocols

## Airplay via shairport-sync

To serve with the Airplay protocol, I
installed the `shairport-sync` package. I will be running one instance
of this per endpoint, which require modifying the init script to start
multiple instances.

    apt-get install sharport-sync

### over ALSA

Run a small instance of shairport-sync with, e.g.:

    shairport-sync -o alsa -a lab -p 5000 -- -d lab

Now, I ran into a problem with this when pulse used libalsa as its
backend. Data was going from shairport-sync through libalsa to pulse
to libalsa again and getting stuck, somehow?

I switched to using Pulse to talk directly to kernel alsa and
configured libalsa to talk to pulse.

#### forwarding virtual outputs from ALSA to Pulse

Even when using the libalsa backend, I am creating my multiple virtual
outputs using Pulse, because it can correct for clock drift. So for
multiple-outputs, my `asound.conf` forwards those sinks to Pulse,
which forwards them back to Alsa. See [above][#relaying-to-pulse].

But this caused a problem with using shairport-sync. The connection
kept starting and dropping. Something in the chain of shairport-sync
-> libalsa -> pulseaudio -> libalsa(again) -> kernel seemed to be
messing up its.

Using Pulse as the system's backend seemed to improve this; now the chain
goes shairport-sync -> libalsa -> pulseaudio -> kernel.

### shairport-sync over Pulse

But shairport-sync also uses a pulseaudio backend; that would get
libalsa the whole way out of the way.

But, the "pa" backend on shairport-sync accepts no arguments. How do I
tell it what output to use? Set environment variables that Pulse looks at?

### running shairport-sync as a service

Having determined arguments that `shairport_sync` worked with,
I placed the desired arguments in `/etc/default/shairport_sync`:

    DAEMON_ARGS="-o alsa -a lab -p 5000 -- -d lab"

### running several instances of shairport-sync as a service

#### check if your distro uses init.d or systemd

I tried messing with `/etc/init.d/shairport-sync` but I could not figure
out how to even emit log messages from it; I eventually concluded it
isn't being run at all, as my distribution uses `systemd`. Delete this
file and look in `/lib/systemd/system/shairport-sync.service`.

#### make sure to --purge when you uninstall an apt package

BTW. After messing up the files in `/etc/init.d` and `/etc/default`
associated with the `shairport-sync` package, I found that if I merely
`apt-get remove --purge` it doesn't remove those files. And if I
delete them manually, the files are not replaced when you delete a
file "owned" by a package, it will not be replaced when you reinstall
the package, unless you --purge the package. How weird is that?

#### running multiple instances using a template service.

Turns out systemd has a way to handle multiple instances of a daemon,
by defining a [template service][].

[template service]: https://www.stevenrombauts.be/2019/01/run-multiple-instances-of-the-same-systemd-unit/

I implemented this by first defining an array of arguments in
`/etc/default/shairport_sync`. I created a file
`/etc/systemd/system/shairport-sync@.service` which was a copy of the original
file. The `@` at the end means `systemd` treats it as a template, so
that when you run,

    systemctl daemon-reload
    systemctl start shairport-sync@cave

then `cave` will be substituted for each `%i` in the template.

    ...
    Description=ShairportSync AirTunes receiver on %i
    ExecStart=/usr/bin/shairport-sync DAEMON_ARGS_%i
    ...

The next problem is to pass each instance's arguments. I created
separate variables in `/etc/default/shairport-sync` for this. Then I
created a master service that controlled all the instances, in a file
`/etc/systemd/system/shairport-sync.target`

This worked for all the direct ALSA ouputs -- but when it comes to the
outputs configured to send from ALSA to Pulse, I get permissions
errors:

    Sep 10 05:31:11 localhost shairport-sync[30345]: ALSA lib pulse.c:242:(pulse_connect) PulseAudio: Unable to connect: Connection refused
    Sep 10 05:31:11 localhost shairport-sync[30345]: alsa: error -111 ("Connection refused") opening alsa device "LR_K_O".

This did not happen when running shairport-sync manually. This may be
because the service is running as the "shairport-sync" user while the
daemon is running as my login user. It needs to have permissions to
talk to pulseaudio as that user. [See above][#permissions].

### can't play multiple streams?

Play mpg123 to one output, then in a separate process play it to
another. Why is it stuttering? If I use the mpg123's pulse output
instead of libalsa, it even crashes pulseaudio for that channel,
yay. What's going on?

## play via PulseAudio on Linux

I want to play from another Linux desktop. How do I get PulseAudio
sinks to appear in my laptop's control panel? Or for that matter how
to play to an RAOP sink?

Server already has the `load-module module-zeroconf-publish` checked.
Are the services discoverable? `avahi-discover` shows that there is a
pulseaudio server `pulse@speakers`.

I installed `paprefs` on the client but the checkboxes for network
access are grayed off. Is that really a 12 year old issue?

So how do I make my own pulseaudio instance see it as an option?
`/etc/pulse/default.pa` exists. I should copy it to `~/.config/pulse`

```
cp /etc/pulse/default.pa .config/pulse/default.pa
```

I then uncommented the lines:

```
load-module module-esound-protocol-tcp
load-module module-native-protocol-tcp
load-module module-zeroconf-discover
load-module module-raop-discover
load-module module-null-sink sink_name=rtp format=s16be channels=2 rate=44100 sink_properties="device.description='RTP Multicast Sink'"
load-module module-rtp-send source=rtp.monitor
```

then `pulseaudio -k` Did this restart? where's the log? Actually that worked great, I now se 

## DLNA/uPNP rendering

`[gmrender-resurrect][]` is a DLNA renderer for Unix. Install
following [these instructions][] and [also these][]

[gmrender-resurrect]: https://github.com/hzeller/gmrender-resurrect/
[these instructions]: http://blog.scphillips.com/posts/2014/05/playing-music-on-a-raspberry-pi-using-upnp-and-dlna-v3/
[also these]: https://rootprompt.apatsch.net/2013/03/07/raspberry-pi-network-audio-player-pulseaudio-dlna-and-bluetooth-a2dp-part-2-dlna/

Example command line:

    /usr/local/src/gmrender-resurrect/src/gmediarender -f "living_room" -u "92cc7113-d3e4-459f-a67c-e8e9bdc951d8" --gstout-audiosink=pulsesink --gstout-audiodevice=lab --logfile=/dev/stdout

So I should be able to go to my Windows Media player and cast a track to gstreamer. This gives:

    ERROR [2020-11-01 04:35:09.582136 | gstreamer] sink: Error: Failed to connect: Access denied (Debug: pulsesink.c(614): gst_pulseringbuffer_open_device (): /GstPulseSink:sink)

After I futzed with some group settings...

##### TODO (gstreamer)

## Eliminating clicks and gaps

Audio requires a lower, more predictable latency from the operating
system kernel than the server loads Linus thinks of as "real
workloads." So even though full preemption is baked into the Linux
kernel, it's

The kenel shipped with Armbian does not have the realtime scheduler
enabled. To check if your kernel had realtime scheduler, run `uname
-a`; the word `PREEMPT` should appear in the response.

### Cross compiling a realtime kernel

Armbian wants you to cross-compile kernels using a [toolchain][] that
runs on x64. This process actually worked pretty smoothly, so I am
leaving it as a manual step. When the build is done, the `output/debs`
directory will contain `.deb` packages you can copy over to your ARM
device and install with `dpkg -i`.
[toolchain]: https://docs.armbian.com/Developer-Guide_Build-Preparation/

In the kernel config menu, the setting you want to enable is
`CONFIG_PREEMPT=y`. as well as the high resolution timer
(`CONFIG_CONFIG_HIGH_RES_TIMERS=y`) which was already enabled.

And -- *this one cost me a day or two of frustrated troubleshooting, * --
-- it seems you need to specifically [disable][]
`CONFIG_RT_GROUP_SCHED`, unless you want to [jump through][] a whole
[bunch of hoops][] to put your RT process into a cgroup and give the
cgroup a priority.
[disable]: https://forum.armbian.com/topic/16489-config_rt_group_schedy-harmuflull-for-real-time-applications/
[jump through]: https://stackoverflow.com/a/60665456/1188636
[bunch of hoops]: https://mjmwired.net/kernel/Documentation/admin-guide/cgroup-v1/cgroups.rst

If you have `CONFIG_RT_GROUP_SCHED` enabled, all your programs will
get "permission denied" when they try to enable realtime, and you
might fruitlessly mass with ulimit and limits.conf but all your
googling might not turn up that notice that there's a third
independent security mechanism, "cgroups," in your way. (but we're not
quite done with [cgroups][]...)
[cgroups]: https://www.freedesktop.org/wiki/Software/systemd/MyServiceCantGetRealtime/

In Armbian the kernel build process makes `.deb` packages in
`output/debs`. After installing them, be sure to mark those packages
to be held, so that a future `apt-get upgrade` doesn't clobber them.

    sudo apt-mark hold armbian-config armbian-firmware-full \
        linux-dtb-current-sunxi linux-headers-current-sunxi \
        linux-image-current-sunxi linux-source\
        linux-u-boot-current-orangepiplus2e

Some more tips for optimizing a kernel for low-latency loads are [here][tips].
[tips]: https://rigtorp.se/low-latency-guide/

Beyond that, if you search for advice on Linux kernel configuration
you'll see a lot of different advice of different ages. Current advice
seems to be that changing the scheduling interrupt frequency
(`CONFIG_HZ_1000`) [doesn't help][HZ-1000] RT processes (though it may
help reduce jitter in non-RT processes.) The tickless scheduler
(`CONFIG_NO_HZ_FULL=y`) [may be useful][no-hz], but only if you jump
through some cpu affinity hoops so that the realtime process has
exclusive run of a CPU core. (TODO: return to this)
[HZ-1000]: https://github.com/raboof/realtimeconfigquickscan/issues/4
[NO-HZ-FULL]: https://www.kernel.org/doc/html/latest/timers/no_hz.html

### Testing the realtime kernel.

[Cyclictest][] is a utility to benchmark the kernel's low latency
performance. I installed it via `apt-get install rt-tests`. This
 [guide][] to Linux audio suggests this command line, run as root:
[cyclictest]: https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start
[guide]: https://wiki.linuxaudio.org/wiki/system_configuration#cyclictest

    # cyclictest -t1 -p 80 -i 10000 -l 10 -m

The output doesn't matter yet -- what does matter is that it doesn't
complain about an "operation not permitted".

### Granting realtime privileges to audio group

When not running as root, `cyclictest` reports:

    WARN: open /dev/cpu_dma_latency: Permission denied
    FATAL: timerthread-1: failed to set priority to 80

[To fix the "permission" error][permissions], create a file
`/etc/security/limits.d/audio.conf` with the lines:
[permissions]: https://jackaudio.org/faq/linux_rt_config.html

    @audio   -  rtprio     95      # out of 99
    @audio   -  nice       -19     # always leave one for root
    @audio   -  memlock    unlimited

This setting should take effect for future logins.  Check this by
starting a new login (as a user in the `audio` group) and running
`ulimit`:

    peter@speakers:~$ ulimit -e -r -l
    scheduling priority             (-e) 40
    real-time priority              (-r) 95
    max locked memory       (kbytes, -l) unlimited

After logging out and logging in again, assuming your user is in the
`audio` group, `cyclictest` should work.

### Making Pulseaudio use realtime mode;

I've configured high priority and/or realtime mode in `/etc/pulse/daemon.conf`:

    high-priority = True
    nice-level = -11
    realtime-scheduling = True
    realtime-priority = 50

Testing this by starting pulse from a user account should give:

    I: [alsa-source-USB Audio] util.c: Successfully enabled SCHED_RR scheduling for thread, with priority 9, which is lower than the requested 50.

### Making realtime mode work from systemd service

While playing with the `pulseaudio.service` file, I keep a window open
on syslog and restart with:

    sudo systemctl stop pulseaudio.socket pulseaudio && sudo systemctl daemon-reload && systemctl start pulseaudio.service

And I probe the capabilities of the running process like:

    getpcaps $($FINDPULSE)
    cat /proc/$($FINDPULSE)/status

One thing I learned here was `cat /proc/$($FINDPULSE)/status` and
check that `NoNewPrivs: 0`. If not, check the [systemd
manual][nonewprivs] and make sure that your systemd entry doesn't do
any of the things that imply `NoNewPrivs:1`.
[nonewprivs]: https://www.freedesktop.org/software/systemd/man/systemd.exec.html#NoNewPrivileges=

So I added the lines:

    RestrictRealtime=no
    MemoryDenyWriteExecute=no
    RestrictNamespaces=no
    LockPersonality=no
    NoNewPrivileges=no

### My service still can't get realtime!

Still, I found this in syslog:

    pulseaudio[12201]: setrlimit(RLIMIT_NICE, (31, 31)) failed: Operation not permitted
    pulseaudio[12201]: setrlimit(RLIMIT_RTPRIO, (9, 9)) failed: Operation not permitted
    pulseaudio[18375]: Failed to acquire high-priority scheduling: No such file or directory

[This page][cgroups] on `freedeskdop.org` suggests that this is a
cgroups problem, even though I disabled the cgroups realtime scheduler
in my kernel. Systemd puts processes into a slice for each service,
and the slice must have some realtime assigned to it. The page
suggests adding some `ControlGroup` settings to the service file:

    ControlGroup=cpu:/
    ControlGroupAttribute=cpu.rt_runtime_us 450000
    ControlGroupAttribute=cpu.rt_period_us  500000

But this resulted in:

    systemd[1]: /etc/systemd/system/pulseaudio.service:35: Unknown key name 'ControlGroup' in section 'Service', ignoring.
    systemd[1]: /etc/systemd/system/pulseaudio.service:36: Unknown key name 'ControlGroupAttribute' in section 'Service', ignoring.
    systemd[1]: /etc/systemd/system/pulseaudio.service:37: Unknown key name 'ControlGroupAttribute' in section 'Service', ignoring.

It seems that since the `freedesktop.org` page was penned, linux now
has moved to [control groups v2][] and [systemd][] thus has [a new
set][systemdv2] of config options.
[control groups v2]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
[headsup]: https://lwn.net/Articles/555923/
[systemdv2]: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html

("Control groups v2" and "unified resource hierarchy" are the same
thing, I think.) At this point, it helped to read something about
cgroups [written in 2020.][2020]
[2020]: https://www.redhat.com/sysadmin/cgroups-part-one

I see in my `/sys/fs/cgroup/unified` directory exists which means we
are in the "hybrid" situation described on [this page][cgroup].
[cgroup]: https://systemd.io/CGROUP_DELEGATION/

In any case, the secret sauce was adding explicit settings for RTPRIO
and nice limits to `/etc/systemd/system/pulseaudio.service`. The
relevant settings are documented on [this page][exec].
[exec]: https://www.freedesktop.org/software/systemd/man/systemd.exec.html

    # for control groups v2:
    LimitRTPRIO=95
    LimitRTTIME=500000
    LimitNICE=-19


### Stuttering when starting an airplay stream

This happens for a few seconds while starting an audio stream.
when playing from my phone, not when playing from my
laptop. Are there any logs from shairport-sync? I enable logging in
`etc/shairport-sync`:

    log_verbosity = 3; // "0" means no debug verbosity, "3" is most verbose.

and then trace the logging with:

    sudo journalctl -f | grep '\(pulseaudio\|shairport\)'

It seems to be starting and stopping streams constantly. due to a momentary underrun.

What happens if I change shairport-sync's output to pulseaudio?

Same thing, but something suggestive in the logs:

    Jun 06 03:39:33 speakers pulseaudio[11773]: Requesting rewind due to corking

Disabling the "corking" module from seems to make for a bit less stutter.

Oddly enough, this "corking" doesn't appear in the logs when shairport
is using the alsa backend, but the improvement still holds. However,
shairport also seems to work worse with the pa backend.

This seems to be tied to shairport-sync's sync functionality. I'm not
relying on that (PulseAudio should handle syncing its outputs) so I
will disable that. Indeed, this reduces the problem to a single gap
instead of an endless loop of them.

### Reduce cpu load?

The gaps are probably coming from CPU load. If two streams are playing
there are more gaps. It might be useful to play with the buffer
size/number in pulseaudio and see how that affects things.

When playing one stream, pulseaudio is registering 60-70% of a core and
shairport-sync is registering 15%. This is a bit much, I wonder if it
has to do with resampling?

Most media I'm playing will be 44.1 kHz. But I have the output set to
48000 right now? Or not because Pulseaudio is the backend right now.
Let's change that.

Hmm setting it to 44100 didn't reduce the CPU usage. Setting resample
methods to "trivial" reduced usage to 30%. Why is it resampling
though?

Maybe if I disable the useless Avantree card? That takes Pulse down to
about 20% cpu.

How about if I disable output syncing?

## shairport-sync is not starting

After working on the Linux-client setup, I can now see pulse endpoints
from Linux but have stopped being able to see them from my
iphone.
`shairport-sync` services are not appearing in `sudo service --status-all`,
But they _are_ showing as running in the output of `systemctl`.
But they do _not_ show up in `avahi-discover`.

```
peter@speakers:~$ systemctl status -l shairport-sync@_downstairs
...
Nov 13 07:51:10 speakers shairport-sync[31467]: Line 28 of the configuration file "/etc/shairport-sync.conf":
Nov 13 07:51:10 speakers shairport-sync[31467]: duplicate setting name
Nov 13 07:51:12 speakers systemd[1]: Stopping ShairportSync AirTunes receiver on _downstairs...
```

Okay, so this was due to a duplicate key for `_resync_threshold_in_seconds.`

## constant gaps, positive and negative sync errors:

But now I am having constant gaps in playback. Repeated errors of the form:

```
0.028415756 "player.c:2273" Large positive sync error: 8669.
0.196357677 "audio_alsa.c:1674" alsa: underrun while writing 352 samples to alsa device.
0.071517245 "player.c:2146" Player: packets out of sequence: expected: 42344, got: 42377, with ab_read: 42378 and ab_write: 42605.
0.023106624 "player.c:2285" Large negative sync error: -8113 with should_be_frame_32 of 1786841005, nt of 1786761617 and current_delay of 699.
0.000974963 "player.c:2300" Play a silence of 8113 frames.
0.079398696 "player.c:2146" Player: packets out of sequence: expected: 42381, got: 42390, with ab_read: 42391 and ab_write: 42620.
0.097913703 "player.c:2273" Large positive sync error: 3063.
0.145762471 "player.c:2146" Player: packets out of sequence: expected: 42403, got: 42420, with ab_read: 42421 and ab_write: 42649.
0.023240869 "player.c:2273" Large positive sync error: 6416.
0.167959879 "audio_alsa.c:1674" alsa: underrun while writing 352 samples to alsa device.
0.055789092 "player.c:2146" Player: packets out of sequence: expected: 42424, got: 42451, with ab_read: 42452 and ab_write: 42681.
0.020607760 "player.c:2285" Large negative sync error: -6137 with should_be_frame_32 of 1786867095, nt of 1786787665 and current_delay of 2633.
0.001723476 "player.c:2300" Play a silence of 6137 frames.
```

So there are both negative and positive sync errors. I tried increasing `drift_tolerance_in_seconds = 0.002` and it appears to have gone away? But it also went away when I undid that change.

Ah, the problem was this:

    resync_threshold_in_seconds = "0";

It should not have the quotation marks.

    resync_threshold_in_seconds = 0;

Go figure!

## Too many airplay targets in my audio controls

Pulseaudio lists all the airplay targets under ipv4 addresses and then again under ipv6 addresses. How can I make it show less?

I changed `/etc/avahi/avahi-daemon.conf` to have:

    [server]
    use-ipv4=yes
    use-ipv6=no

So far so good.

## Suddenly stops producing audio, though client thinks it is playing

This is fixed by disconnecting and then reconnecting. What is going on? This happens when playing via 

## After an hour or two of playtime, we start getting dropouts.

This is the only relevant message at the current log level...

    Nov 22 05:58:13 speakers shairport-sync[11225]:          9.120892469 "audio_alsa.c:1674" alsa: underrun while writing 351 samples to alsa device.

## I unplugged and replugged my system and now it won't start

I suspect the problem is I don't have things plugged into the sae port different port somehow?

Doing `udevadm monitor` and pluggin/unplugging cards shows the addresses

downstairs:

    KERNEL[4910018.788755] add      /devices/platform/soc/1c1d400.usb/usb8/8-1/8-1:1.0/sound/card1 (sound)

upstairs:

    KERNEL[4910382.380398] add      /devices/platform/soc/1c1b400.usb/usb6/6-1/6-1:1.0/sound/card2 (sound)

bluetooth:

    UDEV  [4910457.628054] add      /devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.3/4-1.3:1.0/sound/card3 (sound)

That's different from what I had in udevadm rules. I just plugged it back into the right port. 

## can't play from Linux / pulase any more?

It's been s while since I touched the setup.

I needed a command to read logs... 

     sudo journalctl -f "_SYSTEMD_SLICE=system-shairport\x2dsync.slice" + _SYSTEMD_UNIT=pulseaudio.service
     
 I also want to increase pulseaudio and shairport's logging level
 
 So I re-run my ansible playbook. It fails with 
 
    Dec 26 00:46:32 speakers pulseaudio[13671]: Module "module-zeroconf-publish" should be loaded once at most. Refusing to load.
    Dec 26 00:46:32 speakers pulseaudio[13671]: Failed to parse module arguments
    Dec 26 00:46:32 speakers pulseaudio[13671]: Failed to load module "module-native-protocol-tcp" (argument: "auth-ip-acl=127.0.0.1;10.0.0.0/24;192.168.0.0/24 auth-anonymous=1 # port 4713"): initialization failed.
    Dec 26 00:46:32 speakers pulseaudio[13671]: Failed to parse module arguments
    Dec 26 00:46:32 speakers pulseaudio[13671]: Failed to load module "module-http-protocol-tcp" (argument: "# port 4714"): initialization failed.

This points to some duplicated lines in `/etc/pulse/default.pa`.

Now to follow the logs on the client side:

    journalctl --user -f "_SYSTEMD_USER_UNIT=pulseaudio.service"

it seems that shairport is accepting the connection and pretending that nothing is wrong. Pulseaudio however isn't logging enough messages on either client or server.

Strange that sound goes right through from iOS but not from Linux.

So to change the log verbosity, I can go to `/etc/pulse/daemon.conf` and add 

    log-level = debug
    
on both client and server. Now, after restarting bothe client and server I find that pulseaudio is behaving like somethign is connecting and sending audio.

I will compare server logs between Pulse client and ios client.I didn't come up with much. Pulseaudio logs on the client side have the message

    "Unexpected response when expecting header: RTSP/1.0 200 OK"
    
Which has one google hit, from a person also experiencing this, with no answers.

So the problem may be with my client computer's shairport implementation. I'm on version 3.3.5 of shairport-sync and github is on version 4.3 so why wasn't that upgraded? Looks like 3.3.5 is the version in Ubuntu. It's now packaged as a Docker image, because it's 2023 and no one make packages for distributions anymore.

So, here we go 

# Dockerizing shairport-sync

Just changed shairport-sync to "docker run shairport-sync" in `/etc/systemd/system/shairport-sync@.service.` Startign service now gives this error:

    Dec 26 07:19:53 speakers docker[28375]: docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdo>

Think this is because user shairport-sync was not in group "docker." Fixed, then:

    Unable to find image 'shairport-sync:latest' locally
    
starting an unmodified image always seems to require the full path. Then I am getting these logs:

    Dec 26 08:47:15 speakers docker[3289]: fatal error: unable to listen on port 319. The error is: "Address in use". NQPTP must run as root to access this port. Or is another PTP daemon -- possibly another instance on NQPTP -- running already?

Odd since I thought I was passing the port number on the command line. Can I reproduce this on the command line?

    docker run -ti --network host --device /dev/snd --name shairport-sync_living_room mikebrady/shairport-sync -o alsa -a living_room -p 5003
    
    docker run -d --network host --device /dev/snd --name shairport-sync_living_room mikebrady/shairport-sync -o alsa -a living_room -p 5003
    
Gives me a

    The container name "/shairport-sync_living_room" is already in use by container ...
    
So I can `docker container prune` to remove stopped containers.

The name collision issue means I should be creating my containers ahead of time. So one container per shairport instance. Which gives me an idea, can the arguments be encoded into the container?

Also it is giving me 

    Loading service file /etc/avahi/services/sftp-ssh.service.
    Loading service file /etc/avahi/services/ssh.service.
    *** WARNING: Detected another IPv4 mDNS stack running on this host. This makes mDNS unreliable and is thus not recommended. ***
    bind() failed: Address in use
    Failed to create IPv4 socket, proceeding in IPv6 only mode

How do I address this issue with ipv4?

More importantly, the container appears on my iPhone, accepts connection and playback, but no sound is coming out. What do pulseaudio logs say?

Okay, on my raspi I don't see anything in syslog at all. Still claims to be playing.

Can I see the container logs?

`docker logs shairport-sync_living_room` shows the startup messages but not the syslog?

Here, this page has a tute for multiple instances

https://github.com/noelhibbard/shairport-sync-docker

I don't follow why the static IP addresses... Is there a way to use DHCP on the virtual hosts?

    fatal error: unable to listen on port 319. The error is: "Address in use". NQPTP must run as root to access this port. Or is another PTP daemon -- possibly another instance on NQPTP -- running already?
    
So it seems like this image also runs its own instance of NQPTP and avahi-daemon.... which I suppose is why noelhibbard's compose file assigned it to a different IP address.

So my setup is to template out a docker-compose file that assigns one IP to each endpoint.

I can ping them but I can't connect to the given ports.. Did the daemon start?

    docker exec -it shairport-sync_lab /bin/sh
    
I don't see a shairport-sync process. On further inspection it seems like the container went up but the shairport-sync process exited early.

### Where exactly is the Pulse socket???

Okay, I tried exec'ing shairport manually in one of the images and saw that it was not connecting to the socket. I see my pulseaudio is making a socket at `/tmp/pulse-server` even though it's configured with `socket=/tmp/pulseaudio.socket` My host pulseaudio instance creates a socket at `/tmp/pulse-server` even though it has
    
    load-module module-native-protocol-unix auth-anonymous=1 socket=/tmp/pulseaudio.socket

This appears to be because the socket created in `/etc/systemd/system/pulseaudio.socket` overrides the conf file (how?). So change it in systemd. `systemd daemon-reloaad`.

### Containers deadlock when started as a group

Now, I have a situation where `docker compose up` doesn't seem to be running shairport-sync?. But `docker start shairport-sync_living_room` does make it start and it shows up on the network. And it plays! So cool.

So now to make them all run at once `docker container start` didn't do it?

On further inspection, all the containers start and function if I start them one at a time. But when brought up as a group together, they get stuck in init, while bringing up nqptp or dbus. 

It seems likely that there is some resource contention happening as these services announce themselves to each other, something is timing out or deadlocking.

For the moment, rather than debugging the Docker image, I decided to work around it in systemd. Each endpoint gets an `execStartPre=/bin/sleep 5` and an `After=` to enforce starting in order. SO each are started one at a time.
