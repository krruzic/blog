+++
title = "Play Audio on Two Pulseaudio HDMI outputs; or Adding New ALSA Devices to Pulse"
date = 2020-01-04
draft = false
type = "post"

[taxonomies]
tags = ["pulseaudio", "linux"]
+++

Pulseaudio's autoconfiguration is very limited and simply selects the first HDMI output per card to create sources with.
What if you have two monitors with speakers on the same card and want to play different simulatenous streams?

<!-- more -->

# PulseAudio and ALSA

What's the difference and why do I need two systems to play audio? No idea. What I do know is pulse is built on top of
alsa and queries alsa for device information. Based on that, we'll need to ask alsa for info on our devices.

```bash
# first list our devices
$ alsa -l
card 0: Sound [HyperX Virtual Surround Sound], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: HDMI [HDA ATI HDMI], device 3: HDMI 0 [HDMI 0]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 1: HDMI [HDA ATI HDMI], device 7: HDMI 1 [HDMI 1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: HDMI [HDA ATI HDMI], device 8: HDMI 2 [HDMI 2]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 1: HDMI [HDA ATI HDMI], device 9: HDMI 3 [HDMI 3]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: Generic [HD-Audio Generic], device 0: ALC1220 Analog [ALC1220 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: Generic [HD-Audio Generic], device 1: ALC1220 Digital [ALC1220 Digital]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

You should get similar output to the above, but with your own card information. On my computer I have a headset plugged
in (`card 0`), my GPU with 4 hdmi ports (`card 1 devices 3,7,8,9`) and my motherboard's audio controller. In my case, I
want to add card 1 device 8 to pulseaudio so I can play sound out of my second monitors speaker. By default pulseaudio
adds HDMI 0 (card 1 device 3).

# Configuring Pulse

Open up `/etc/pulse/default.pa` with your favourite editor as root

```bash
$ sudo vim /etc/pulse/default.pa
# add this line to the top of the file, changing 1 and 8 to match your selected
# card and device. Set the description and sink_name to whatever you'd like
load-module module-alsa-sink device="hw:1,8" sink_properties="device.description='Secondary HDMI'" sink_name=second_hdmi
```

Now that you have your device added you'll need to restart pulse. Once restarted you should see your newly added device
in whatever GUI audio control program you use.

```bash
$ pulseaudio --kill; sleep 1; pulseaudio --start
```

# Issues

One new issue I've encountered on a newer version of pulse is that adding a line that references the same card as a card
automatically added by pulse (ex. adding a second HDMI output device) will delete the auto-configured device. To fix
this, just add another line to `default.pa` referencing your auto-configured device.

# Sources

[Arch Wiki](https://wiki.archlinux.org/index.php/PulseAudio/Examples#Simultaneous_HDMI_and_analog_output) - Discusses
playing the same audio over two outputs
