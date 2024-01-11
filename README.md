# House Multizone Audio system

This is an Ansible role that configures a server to drive multiple
outputs of several soundcards, for use in a multi-room home audio system.

With this setup I can direct audio from my phone to any room, as well
as to virtual targets that play simultaneously in several rooms.

Supports PulseAudio, AirPlay, and maybe soon other protocols
(DLNA/Chromecast)

The machine I'm using for this setup is an Orange Pi Plus 2e with a
pair of USB2-connected, 7.1 channel DACs, running Armbian Focal.

Also in this repo is a detailed [troubleshooting log][notes.md],
because in 2020, setting up audio on Linux is still a mess, and a lot
of information you find searching online is woefully out of date.

# Update

In 2023, no one makes packages for distributions anymore. In order to upgrade shairport-sync without compiling from scratch we have to learn Docker.

There's a weird issue with the resulting setup, in that if I start more than 4-5 containers at once, the images lock up while starting dbus or ntpd.

My solution was to start them one at a time, staggering the starts using systemd `execStartPre=/bin/sleep 

Note that to bring down the service you need to run `systemd stop
shairport-sync.target`. If you use Docker to stop the images systemd
is just going to restart them.
