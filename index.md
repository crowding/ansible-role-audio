# House Multizone Audio system

This is an Ansible role that configures a server to drive multiple
outputs of several soundcards, for use in a multi-room home audio system.

With this setup I can direct audio from my phone to any room, as well
as to virtual targets that play simultaneously in several rooms.

Supports PulseAudio, AirPlay, and maybe soon other protocols
(DLNA/Chromecast)

The machine I'm using for this setup is an Orange Pi Plus 2e with a
pair of USB2-connected, 7.1 channel DACs, running Armbian Focal.

Also in this repo is a detailed [troubleshooting log][motes.md],
because in 2020, setting up audio on Linux is still a mess, and a lot
of information you find searching online is woefully out of date.
