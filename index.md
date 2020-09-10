---
layout: page
title: "pi-audio"
date: 2014-03-24 21:29
comments: true
sharing: true
footer: true
---

# House Multizone Audio system using ARM devboard.

## Or, how to run a bunch of USB audio DACs at once and serve them over the network

This is a revamp of my earlier setup, but with a newer Orange Pi board.

I want to be able to stream music from any mobile device (laptop,
iDevice, Android-, guest's phones) to any room in the house. I also
want to be able to take the podcast I'm listening to upstairs and
move it downstairs, or to other rooms as I move around.

I also want to be able to connect my Bluetooth headphones to it, and
have audio routed there automatically, and be able to walk around the
property (so a long range transmitter)

This should also support me wandering all the way around with my
headphones. (perhaps I should be able to connect my phone as a client to the
stream?)

I might wish there to be a web management console to connect the streams.

Now, there are already tutorials for multiroom audio on the Raspi, but
they require you to have a Raspi for each endpoint. My house's layout
is such that you can reach almost every room from a location near the
furnace, so it makes a lot of sense to have one central server with
several analog outpits. This wafes theexpense and config hassle.

This is also because the DACs that come on these devboards are all kind
of crap.

For these instructions I am running on Armbian Wheezy install.

## Hardware Setup

* one [Orange Pi Plus 2E][orangepi] running armbian. This is based on
  the Allwinner H3 chip.
  * I had originally tried this with a Raspberry Pi, but ran into
    problems with its USB implementation. The Rpi had two USB2.0
    ports, but on closer inspection these two ports were shared by one
    bus, and worse, it was a passive hub.

    A passive hub is a problem because many USB DACs run at USB 1.0 speed;
    which means that while talking to the
* [One 7-port USB 2.0 hub][dlink]. This was initially required because of
  the Raspi's USB shenanigans (i.e. use a proper active hub to better while
  the Raspberry Pi offers three or four USB ports,
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

Started over with a fresh Debian install -- with I was able to
install Debian on a 2GB microSD card.

This install seems to be using my native resolutions just fine (on the
console, haven't tried with Xwindows yet.)

## Software setup.

This time I'll be using Ansible to make this setup reproducible.

I have Windows, Mac and Linux devices on my home network, wo the solution should.
Here's a short list of network sound protocols:

* Airplay is native on Mac and iOS devices
* PulseAudio is standard on Linux
* JACK, another Linux option aimed at more technically
  exacting setups
* Miracast?
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

Poke around in `/sys` and you may find some more generic information to

Using this information, you can now write a file names something like
`/etc/udev/rules.d/39-audio.rules` which would look something like
this:

```
SUBSYSTEM!="sound", GOTO="sound_end"
ACTION!="add", GOTO="sound_end"
DEVPATH=="/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.3/4-1.3:1.0/sound/card?", ATTR{id} = "living-room"
DEVPATH=="/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.3/4-1.3:1.0/sound/card?", ATTR{id} = "garage"
DEVPATH=="/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.2/4-1.2:1.0/sound/card?", ATTR{id} = "lab"
DEVPATH=="/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.2/4-1.2:1.0/sound/card?", ATTR{id} = "kitchen"
DEVPATH=="/devices/platform/soc/1c1c000.usb/usb4/4-1/4-1.5/4-1.5.2/4-1.5.2:1.0/sound/card?", ATTR{id} = "patio"
LABEL="sound_end"
```

The udev rules can also match on the vendor and brand of the device,
whiuch might be more convenient, if you don't have devices with identical
chipsets.

I actually generate this file from Ansible via a [template][],
generated from the corresponding [hostvars file][] where I collect information
about the setup in one place.

[template]: ~/.ansible/templates/audio_udev_rules.j2
[hostvars file]: ~/.ansible/host_vars/speakers.local.yaml

After writing the rules file, apply the rules using `udevadm control
--reload-rules && udevadm trigger`, then reconnect your devices. Then
you should be able to see the names in the output of `aplay -l` as well as in 
the directory `/proc/asound/`.

## Testing

Playing from the command line to a specific device:

```
mpg123 -o alsa -a lab avril.mp3
mpg123 -o alsa -a lab avril_14th.mp3
```

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

Instead, we relay the output from ALSA to PulseAudio, (which will send
its data back to ALSA, in the present configuration, but that's
neither here nor there. Do this using the alsa pulse plugin, for example:

    pcm.everywhere { type pulse; device "everywhere" }

# Installing Pulse

Last time the above is what I did to make multiple outputs. This time
I want to see if PulseAudio can help. Install pulse with ```apt-get
install pulseaudio-module-zeroconf``` (which will also install the main
pulseaudio service.)

In previous attempts with a Raspberry Pi, I failed to get pulse
PulseAudio. It would try to talk to ALSA and then ALSA would talk back
to pulseAudio, or something....

But based on [this blog entry][blog] (by one of the authors of PulseAudio), I
think that it matches my use case better than JACK. So I might try again.
[blog]: http://0pointer.de/blog/projects/when-pa-and-when-not.html

List your audio devices with ``pacmd list-cards``.

This told me that all the cards I'd named and configured in ALSA
configs didn't carry over -- they have some complicated USB path name
that that gained from scanning ` Noe, when I do a `pacmd list-cards`
it sees both the ALSA hardware cards, but it doesn't see the virtual
plugs. I thin I will recapitulate those in pulseaudio land anyway. But
more worrisome is, it doesn't see the card names that I set in udev
rules?

It looks like by default, pulseaudio interprets the contents of udev
by its own rules to make sound cards. This is controlled in
`/etc/pulse/default.pa` I commented out the lines about automatic
hardware detection and added:

    load-module module-alsa-sink device=surround7 tsched=0
    load-module module-alsa-sink device=surround5 tsched=0

(but I originally did without `tsched=0`, which [led to a lot of chops
and dropouts][dropouts].) When changing pulse configuration, restart Pulse with
the commands `pulseaudio -k; pulseaudio -v -D`.
[dropouts]: https://wiki.archlinux.org/index.php/PulseAudio/Troubleshooting#Choppy_sound_with_analog_surround_sound_setup

## Testing Pulse outputs

To play to a Pulse output:

    mpg123 -o pulse -a alsa_output.surround7 avril.mp3

## Splitting outputs

To split a surround sound device into virtual stereo devices, I wrote
lines like the following in `default.pa`

    load-module module-remap-sink sink_name=laundry sink_properties="device.description='laundry'" remix=no master=alsa_output.surround5 channels=2 master_channel_map=front-left,front-right channel_map=rear-left,rear-right

Test with:

    mpg123 -o pulse -a laundry avril.mp3

### My outputs are getting redirected to the default!

I was having problems with several outputs reing redirected to the
first output. I traced this to the line placed at the end of default.pa:

    set-default-sink lab

which appeared to _redirect_ the sinks that were defined earlier in
the file. I'm not sure what the reason for this behavior is. The
evidence was lines like this in the output of `pulseaudio -v`:

    I: [pulseaudio] sink.c: The sink input 3 "(null)" is moving to lab due to change of the default sink.

Placing the `set-default-sink` ahead of the definitions didn't work; the
solution was to place the `set-default-sink` after the first one was
defined. I also disabled the line:

    load-module module-default-device-restore

### ALSA and Pulse channels don't map the same.

Note that this uses channel "names" rather than channel outputs. In
Pulse the surround channels come in a stereotyped order. Are they the
same order as ALSA exposes?

Given both alsa and pulse names for the outputs, I should check if they map
the same. They do not. Let's see.

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

For the present purposes I fixed the mapping in the way I am
genrerating `default.pa`.

## Combining different outputs into one virtual sink.

Now you want to play a single source over multiple sinks. This can be
setup in one line like this:

    load-module module-combine-sink sink-name="LR_K_O" slaves="kitchen,office,living_room" channels=2

Once I spelled everything right, this actually does start to work a
bit better than the ALSA.

## [TODO] Virtual sink to the garage?

In the garage, which isn't physically wired to the rest of the house,
I have an Apple Airport Express -- an ancient one that
[modern MacOS refuses to configure][express], even. It has an airplay
output. Can I add it to this system? What I want is for PulseAudio to
discover the device and add it to the available sinks.

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
small sound card of its own.

## Web control panel?

Now the nice thing would be if there could be a web interface that
shows the matrix of what is playing to where. Alas, mostly we just
have web interfaces to a volume control.

https://github.com/Siot/PaWebControl

# Network audio servers

## Airplay via shairport-sync

To serve with the Airplay protocol, I
installed the `shairport-sync` package. I will be running one instance
of this per endpoint, which require modifying the init script to start
multiple instances.

    apt-get install sharport-sync

### over ALSA

Run a small instance of shairport-sync with, e.g.:

    shairport-sync -o alsa -a lab -p 5000 -- -d lab

#### low volume?

At first this appeared to play nothing, but I found it is actually
playing very softly.

This problem went away, not sure why. Running pulsemixer? A fix in
ctl block in alsamixer?

#### forwarding virtual outputs from ALSA to Pulse

If I use ALSA, this will require driving the virtual multiple outputs
from ALSA. I had [earlier][#multiple] decided to create the virtual
outputs using Pulse, which is capable of correcting for clock skew. I
instead opted to crewate shadow virtual devices that pipe from ALSA to
Pulse, which handles synchronization between devices better, and then
sends back to ALSA. See [above][#relaying-to-pulse].

### over Pulse

Meanwhile if I try running it over Pulse, as in `shairport-sync -o pa ...`
it appears to connect and play, but nothing is coming out the speakers
when playing from my mac. There is a Github post alleging that
shairport-sync and pulse don't play well. But I earlier chose to skip
over ALSA when creating multiple outputs. A workaround might be to
create virtual outputs in ALSA that redirect to Pulse.

There is a question of whether to make Shairport use ALSA directly or
layer it on top of Pulse. So far it seems to work better with ALSA.

### running shairport-sync as a service

Having determined arguments that `shairport_sync` worked with,
I placed the desired arguments in `/etc/default/shairport_sync`:

    DAEMON_ARGS="-o alsa -a lab -p 5000 -- -d lab"


### running several instances of shairport-sync as a service.

#### don't use init.d

I tried modifying `/etc/init.d/shairport-sync` but I could not even
figure out how to even emit log messages from it. I think init.d
scripts are actually deprecated these days and are being redirected to
another mechanism. You actually need to go look in
`/lib/systemd/system/shairport-sync.service`.

#### make sure to --purge

BTW. After messing up the files in `/etc/init.d` and `/etc/default`
associated with the `shairport-sync` package, I found that if I merely
`apt-get remove --purge` it doesn't remove those files. And if I
delete them manually, the files are not replaced when you delete a
file "owned" by a package, it will not be replaced when you reinstall
the package, unless you --purge the package. How weird is that?

#### running multiple instances using a template service.

Turns out systemd has a way to handle multiple instances, by defining
a [template service][].

[template service]: https://www.stevenrombauts.be/2019/01/run-multiple-instances-of-the-same-systemd-unit/

I implemented this by first defining an array of arguments in
`/etc/default/shairport_sync`. I created a file
`/etc/systemd/system/shairport-sync@.service` which was a copy of the original
file. The `@` at the end means systemd treats it as a template, so
that when you run

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

## Miracast

I believe the nearest Windows/Android/Linux equiavlent to Airplay is
Miracast. So let's see if we can set that one up too. There's
[MiracleCast][] which might be for doing screencasting? for just
audio, there's



[miraclecast]: https://github.com/albfan/miraclecast
