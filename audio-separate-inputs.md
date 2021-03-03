# How to separate two inputs in Analog Stereo Audio?

Okay... first of all: when Pulseaudio starts, It loads `/etc/pulse/default.pa` which says "detect the cards automatically for me and map them.". And then that's done, but wrong way (at least in my case).

When there are 2 inputs, you MUST SEPARATE, IT DOESN'T MATTER IF THEY ARE THE SAME CARD. That's what the automatic detection doesn't does.

So what we need to do is basically set something to stop this detection, and set ourselves what we want.

If you're not familiar with terminals, and etc., don't worry, I'll explain every command I do.

## Requirements

- A terminal
- A text editor
- Pulseaudio
- Something to test your audio WITH PULSEAUDIO.
  - IMPORTANT! It must use **PULSEAUDIO** DEVICES, NOT **ALSA** ONES! It means arecord and aplay will NOT work here.
  - I recommend paplay, as It uses pulseaudio and not alsa.

## Step 1 - copy the pulse configuration file and get some information

### Copying pulse configuration

Open your terminal and copy the file to your home location with

```bash
cp /etc/pulse/default.pa ~/.config/pulse/default.pa
```

The copied file will override the original one on the next PulseAudio launch.

### Getting information

#### Getting all devices

Then use this command:

```bash
arecord -l && aplay -l
```

It may show something like this, but with more or less things:

```
**** List of CAPTURE Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC887-VD Analog [ALC887-VD Analog]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 2: ALC887-VD Alt Analog [ALC887-VD Alt Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
**** List of PLAYBACK Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC887-VD Analog [ALC887-VD Analog]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
```

Look that there's always something like this:

```
card <card number>: <card name>, device <device number>: <device name>
	Subdevices xxxx
	Subdevice xxx
card <card number>: <card name>, device <device number>: <device name>
	Subdevices xxxx
	Subdevice xxx
[...]
```

So take all the card and device numbers.

In the example, I have card 0, with device 0 and 2 for capture and 0 for playback.

So I have:

``````
Capture Devices
Card 0 (name: PCH), Device 0
Card 0 (name: PCH), Device 2

Playback Devices
Card 0 (name: PCH), Device 0
``````

Don't mind if some capture devices have the same card and device number as playback ones, ALSA separates them.

#### Getting the sample rates for the devices.

This was important in my case, as the sample rates were different. This command should do the trick:

```
pacmd list | grep -v "module-" | egrep "sample spec|name:|alsa\.card"
```

For me It shows me something like this:

```
Default sample spec: s16le 2ch 44100Hz
Default sink name: alsa_output.pci-0000_00_1f.3.analog-stereo
Default source name: alsa_input.pci-0000_00_1f.3.analog-stereo
        name: <alsa_output.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
                alsa.card_name = "HDA Intel PCH"
        name: <alsa_output.pci-0000_00_1f.3.analog-stereo.monitor>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
                alsa.card_name = "HDA Intel PCH"
        name: <alsa_input.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
                alsa.card_name = "HDA Intel PCH"
        name: <alsa_card.pci-0000_00_1f.3>
                alsa.card = "0"
                alsa.card_name = "HDA Intel PCH"
```

We only need to pick the sample rate for each device. To get things easier, I just put the names of the outputs and inputs upside their sample rates and their card numbers on my command, so just search for `input` or `output` and take the next line rate, and the other line `alsa.card`. Example:

```
        name: <alsa_output.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
Our output has 44100Hz of sample rate, and It seems to be the card 0
        name: <alsa_input.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
Our input has 44100Hz of samplerate, and It seems to be card 0 also.
```

And also, ignore the monitor ones

**Important!** DON'T FORGET to name the output rates names. We have both input and output with same sample rate, but Imagine they don't. 

**IMPORTANT!** If there are more devices, 

## Step 2 - Configure the modules again.

Search in the copied file (`~/.config/pulse/default.pa`) for `### Load audio drivers statically`

This part is commented with #, so It does not run. What we need is to put something like this (the next example is in my case):

```bash
### Load audio drivers statically
### (it's probably better to not load these drivers manually, but instead
### use module-udev-detect -- see below -- for doing this automatically)
load-module module-alsa-sink device=hw:0,0 sink_name=sound_output
update-sink-proplist sound_output device.description="Sound Output"

load-module module-alsa-source device=hw:0,0 source_name=input1 rate=44100
update-source-proplist input1 device.description="Input One"

load-module module-alsa-source device=hw:0,2 source_name=input2 rate=44100
update-source-proplist input2 device.description="Input Two"
```

okay, I think It's way too fuzzy at a first view, but let's explain what's happening.

```bash
load-module module-alsa-sink device=hw:0,0 sink_name=sound_output
# on this first line we are loading the 'module-alsa-sink' module to create a sink which takes the card 0, device 0 as the Playback device. Then we put the new sink name (not yet the display name) and voila! We have a new output!
update-sink-proplist sound_output device.description="Sound Output"
# on this other line we just set the sink display name as "Sound Output" :)

load-module module-alsa-source device=hw:0,0 source_name=input1 rate=44100
# again, here we load another module, 'module-alsa-source' which creates a source that takes the card 0, device 0 as the Capture device, and setting the sample rate as 44100. Then we put again a name (which is not the display name). Now we have a new input!
update-source-proplist input1 device.description="Input One"
# on this other line we just set the sink display name as "Input One" :)
```

Then search for lines like this:

```bash
### Automatically load driver modules depending on the hardware available
.ifexists module-udev-detect.so
load-module module-udev-detect
.else
### Use the static hardware detection module (for systems that lack udev support)
load-module module-detect
.endif
```

And put them like this:

```bash
### Automatically load driver modules depending on the hardware available
#.ifexists module-udev-detect.so
#load-module module-udev-detect
#.else
### Use the static hardware detection module (for systems that lack udev support)
#load-module module-detect
#.endif
```

## Step 3 - Restart the daemon

If everything go fine, when restarting the daemon you should have either audio or no inputs working at all, but at least output audio. All your opened applications should have no sound for now, and you may need to restart them.

To restart It:

```bash
pulseaudio -k
# If your Pulseaudio don't start again, do:
pulseaudio -D
```

Then we need to mess with alsamixer.

## Step 4 - alsamixer for adjusting everything

### Opening

Okay, now we have everything done on PulseAudio. Now we are going to ALSA, as something could have gone wrong.

To open alsamixer, open a terminal and do:

```
alsamixer
```

You must see something like this:

![](https://i.imgur.com/aMPAddK.png)

So press F6 and change to your card. In my example, my only card is 0. If you have more, this applies to the other ones too.

### Messing up with my card.

In my sound card the only things I needed to change were the Capture devices. The playback one was just fine, but my capture ones were not working. as I would. So I selected the options with arrows left and right and set them with up and down. The ones I needed to change are the ones marked in red.

![](https://i.imgur.com/TCup8m3.png)

But what does these means?

Yes, I know ALSA is not too intuitive, but... basically the options are:

> X Boost - Line gain in dB for any device? Not sure which one, as It identifies with a number.
>
> Capture (X) - Means the volume of the input device for X.
>
> Input Source (X) - Means the source of the input device for X.

In my example, there are 1 and a "not numbered" devices, but I'm still curious about why the gain is not written at least "Device 1 Boost" and "Device Source". I know, It's fuzzy yet, but It's more understandable than the other.

**IMPORTANT!** See that "Capture 1" device haves a red text saying "CAPTURE", right? You see that "Capture" device has no Capture? This was my fault, It SHOULD SHOW "CAPTURE"! 
You do this by pressing space bar on "Capture". DON'T FORGET!

When finished, press Esc and you're done! If necessary, redo the 3rd step again.

## Troubleshotting

#### I followed all the steps until Step 3, and then all audio is gone!

First of all, try to start again the Daemon with `pulseaudio -D`

If your audio is back, but with any problems, continue the guide from Step 4.

If It shows up an error, please, ask for help. It can be any forum or also me.

If you need the audio for now, and don't mind using the old configuration, delete the `~/.config/pulse/default.pa` file and do `pulseaudio -D` again. It should work

#### I just boot up and there is no audio at all, what to do?

Okay, that was a fuzzy problem to me also. What you probably want to do is to change from card 0 to PCH (like my example, which says card 0 is named PCH).