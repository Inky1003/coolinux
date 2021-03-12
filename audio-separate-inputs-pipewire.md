# How to separate two inputs in Analog Stereo Audio (for Pipewire users)?

Pipewire works with other files which not the Pulseaudio ones, as everyone should note, but It uses the same automated system which Pulseaudio uses, so 

## Requirements

- A terminal
- A text editor
- Pipewire

## Step 1 - copy the pulse configuration file and get some information

### Copying pulse configuration

Open your terminal and copy the file to your home location with

```bash
cp /etc/pipewire/pipewire.conf ~/.config/pipewire/pipewire.conf && cp /pipewire/media-session.d/media-session.conf 
```

The copied file will override the original one on the next Pipewire launch.

### Getting information

#### Getting all ALSA plugged devices

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

**12/03/2021 EDIT:** take the card name also. 

In the example, I have card 0, with device 0 and 2 for capture and device 0 for playback.

So I have:

``````
Capture Devices
Card 0 (name: PCH), Device 0
Card 0 (name: PCH), Device 2

Playback Devices
Card 0 (name: PCH), Device 0
``````

Don't mind if some capture devices have the same card name and device number as playback ones, ALSA separates them.

#### Getting the sample rates for the devices.

This was important in my case, as the sample rates were different. (**12/03/2021 EDIT:** I'm sorry to tell that this depends completely on what happens. Sometimes It uses the right samplerate and sometimes It doesn't. Do this just If you're not sure.) This command should do the trick:

```
pacmd list | grep -v "module-" | egrep "sample spec|name:|alsa\.card"
```

For me It shows me something like this:

```yaml
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

```yaml
        name: <alsa_output.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
# Our output has 44100Hz of sample rate, and It seems to be the card 0
        name: <alsa_input.pci-0000_00_1f.3.analog-stereo>
        sample spec: s16le 2ch 44100Hz
                alsa.card = "0"
# Our input has 44100Hz of samplerate, and It seems to be card 0 also.
```

And also, ignore the monitor ones

**Important!** DON'T FORGET to name the output rates names. We have both input and output with same sample rate, but Imagine they don't. 

## Step 2 - Configure Pipewire

### Enabling those devices

Open `~/.config/pipewire/pipewire.conf`

With the PCH card that does have the 0 playback device and the 0 and 2 recording devices, we should make a new Pipewire adapter for them, using ALSA pcm sinks/sources.

Search for a line like this:

```yaml
context.objects = {
    #<factory-name> = {
    #    [ args  = { <key> = <value> ... } ]
    #    [ flags = [ [ nofail ] ]
    #}
# [...]
}
```

Then add these lines after the first opening brace `{` :

```yaml
context.objects = {
	adapter = {		# This Creates a new adapter
        args = {	# With those arguments:
            factory.name    = api.alsa.pcm.sink			# This is an ALSA pcm sink
            node.name       = sound-output				# Named "sound-output"
            node.description        = "Sound Output"	# Which the user can find It by "Sound Output"
            media.class             = Audio/Sink		# This tells pipewire that this is a Sink
            api.alsa.path           = "hw:PCH,0"		# And finally the ALSA device number and card name
        }
    }
    adapter = {
        args = {
            factory.name    = api.alsa.pcm.source		# This is an ALSA pcm source
            node.name       = input-1					# Named "input-1"
            node.description        = "Input 1"			# Which the user can find It by "Input 1"
            media.class             = Audio/Source		# This tells pipewire that this is a Source
            api.alsa.path           = "hw:PCH,0"		# And finally the ALSA device number and card name
        }
    }
    adapter = {
        args = {
            factory.name    = api.alsa.pcm.source		# The next lines are the same as above.
            node.name       = input-2
            node.description        = "Input 2"
            media.class             = Audio/Source
            api.alsa.path           = "hw:PCH,2"
        }
    }
}
```

The parts after the # on this part are added by be to explain what is happening.

### Disabling auto detection

Open `~/.config/pipewire/media-session.d/`

Then we disable auto detection as It would disable our newly created adapters.

Search for this:

```
	alsa-monitor
```

and then comment It with #

```yaml
	# alsa-monitor
```

## Step 3 - alsamixer for adjusting everything

### Opening

Okay, now we have everything done on Pipewire. Now we are going to ALSA, as something could have gone wrong.

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

**IMPORTANT!** See that "Capture 1" device haves a red text saying "CAPTURE", right? You see that "Capture" device has no red-colored "Capture" text? This was my fault, It SHOULD SHOW "CAPTURE"! 
You do this by pressing space bar on "Capture". DON'T FORGET!

When finished, press Esc.

**12/03/2021 EDIT:** Someone told me that after pressing Esc we need to save the settings to ALSA with:

```bash
sudo alsactl store
```

## Step 3 - Restart pipewire

**IMPORTANT!** Sometimes Pipewire bugs or doesn't get the audio back again, so you need to restart your computer. That is no problem, as when you restart the computer everything is back. So **if Pipewire bugs, restart your computer.**

If everything go fine, when restarting pipewire you should have either audio or no inputs working at all, but at least output audio. All your opened applications should have no sound for now, and you may need to restart them.

To restart It:

```bash
systemctl --user restart pipewire
```

Then everything is done :)