# ALSA

---
For ayone interested in ALSA kernel development...
![This is not the presentation you are looking for](./obi_wan.jpg)   <!-- .element: class="fragment" data-fragment-index="1" -->

|||
Userspace ALSA
* What?
* Why?
* Configuration files
  * Link with alsa-lib
* Examples using alsa-lib

---
## What?
* Advanced Linux Sound Architecture
* Software framework that provides a generic API for sound card device drivers
* Replacement for Open Sound System (OSS)

|||
## What?
* ALSA fixed some shortcomings of OSS at the time
  * Simultanous access to sound device was not supported in OSS
  * user-space operations
    * Mixing
    * Loopback/snooping
    * Support for non-interleaved interfaces
* Main reason was OSS moved to a proprietary license
  * Moved back to GPL in 2007
  * Kernel: Once you go ALSA, you never go back   <!-- .element: class="fragment" data-fragment-index="1" -->

|||
## What?
* Automatic configuration of sound devices
  * Through the infamous /usr/share/alsa/cards/*.conf
  * Modularized sound drivers
  * Thread-safe* design
  * User space library (alsa-lib) to simplify programming
    * This is the one we will be using today
  * Backwards compatibility layer with OSS

|||
## History
* Started in 1998
* Developed separately from the Linux Kernel
* Introduced in v2.5 in 2002
  * OSS declared deprecated
* Replaced OSS completely in v2.6
  * There is however still a compatibility layer in ALSA
  * But as you might have guessed, nobody uses it (anymore)
* OSS is still being maintained
  * Targetting Linux and FreeBSD

|||
## Who uses it?
  * Linux Sound Servers
    * PulseAudio
    * JACK
  * PortAudio
    * Backbone of Audacity
  * PipeWire

---
# Why?

Why use ALSA/OSS/sound server over ALSA/OSS/sound server?

|||
## Why you should use ALSA over OSS
  * Only use OSS if you just woke up from a 20 year nap and use kernel v2.4
  * More support for ALSA/better supported in Linux

|||
## Why you should use ALSA over a sound server
* Direct/Full control
  * Can also be a disadvantage
   * You are responsible for xrun recovery
   * You are responsible for ALSA quirks
* Minimal performance overhead
  * Less latency?
    * Not true with JACK
    * But JACK does take more CPU cycles
* Mixing built-in through plugins
    * dmix, dsnoop, asym
	* See https://alsa.opensrc.org/AlsaSharing
	* Don't know about the latency impact of this though...

But it's still a lot of hassle   <!-- .element: class="fragment" data-fragment-index="1" -->

|||
## Why you should use a sound server over ALSA
* Takes care of all the hassle
* Multiple programs accessing same device
* Mixing streams
* Separate stream volume control
* Redirecting streams
* Changing settings on the fly
  * sample format
  * channel count
* Virtual audio devices
  * ALSA devices are linked to a real HW device

|||
## Why use X over ALSA
ALSA documentation is horrible
  * Samples/bytes/frames used interchangebly
    * Even though they aren't!!!
  * Incorrect for some features
    * Asynchronous mode
      * SND_PCM_ASYNC in snd_pcm_open()
  * Job protection?  <!-- .element: class="fragment" data-fragment-index="1" -->

---
## Internals

|||
### Main configuration
* Default config
  * /usr/share/alsa/alsa.conf
  * Not meant to be changed
  * Normally, just defines default devices without any plugins
* System wide specific configuration
  * /etc/asoundrc
* User specific configuration
  * ~/.asoundrc
* All config files are parsed every time an ALSA device is opened
  * No restart necessary for changes to take effect!

|||
### Configuration
Alsa uses *.conf files to configure sound devices
* Located in /usr/share/alsa/cards
* Organized in a hierarchical fashion
  * Cards
    * A physical card or logical kernel device capable of input and/or output
  * Devices
    * A card can have several devices
	* Can be operated independently from each other
  * Subdevices
    * A sound endpoint for the device
	* F.e. Left and right channel can be separate subdevices

|||
### Configuration Pitfalls
* Device can have separate subdevice for left or right channel
  * Can be controlled separately in principle...
  * In practice, writing/reading from a subdevice locks the other subdevices
* Application selects audio device using a device string
  * Device string is the 'absolute' path of the device
    * card,device[subdevice]
  * aplay utility can list all pcm devices
    * aplay -L

|||
### Configuration
Each card sets one/multiple interfaces which determine what ALSA API can be used.
* pcm
  * PCM interface
  * snd_pcm_* API
  * API used to read/write audio data to a device
* ctl
  * Control interface
  * snd_ctl_* API
    * snd_ctl_get_dB_range, set/get parameters from ALSA driver, ...
  * Transfer data in Type/Length/Value(TLV)-way to ALSA driver from userspace
  * Fun fact: No limitation on size of data
    * Abused for I/O operations

|||
More and more interfaces...
* amixer
  * Simple mixer interface
  * snd_mixer_* API
  * Setting volume, ...
* amidi
* seq
* hwdep
* rawmidi
* ...

|||
### Configuration
* User can hook in ALSA plugins to a device
  * Conceptually, an ALSA device is a wrapper for its plugins
  * We built our usecase in the config files

|||
### PCM Configuration
The 'type' field determines which plugin is loaded for this PCM device.
* hw
* plug
* shm
* dmix
* dsnoop
* asym
* softvol
  * If sound card does not support volume control by HW
  * Add pure SW volume control for a device
* ...

See https://www.alsa-project.org/alsa-doc/alsa-lib/pcm_plugins.html for a full list.

|||
### PCM Configuration
#### hw type
  * Direct access to the kernel device
  * No software mixing or stream adaptation support
  * Minimal latency
  * PCM parameter that is not supported by HW?
    * Error is returned

|||
### PCM Configuration
#### plug type
  * "Plug-and-play"
    * Performs channel duplication, resampling, ...
  * PCM parameter that is not supported by HW?
    * No error returned
  * Not something you want if latency is critical

|||
#### Config file example
What does this say?

```bash
# My awesome PCM device
pcm.plug0 {
  type plug
  slave {
    pcm "hw:0,0"
  }
}
```

|||
#### Config file example
```bash [2]
# My awesome PCM device
pcm.plug0 {
  type plug
  slave {
    pcm "hw:0,0"
  }
}
```

ALSA PCM device with name 'plug0'
  * Determines which API we can use to open the device in alsa-lib
  * In this case : PCM

|||
#### Config file example
```bash [3]
# My awesome PCM device
pcm.plug0 {
  type plug
  slave {
    pcm "hw:0,0"
  }
}
```
Load the plug plugin
  * Handles data output to this device
  * Does automatic sample rate conversion if needed
  * Plugins need a slave device!

|||
#### Config file example
```bash [4-6]
# My awesome PCM device
pcm.plug0 {
  type plug
  slave {
    pcm "hw:0,0"
  }
}
```

This PCM device is for your first sound card (0) and first device (0)

|||
ALSA leaves some freedom on how to write the configuration files
```
pcm_slave.slave0 {
    pcm "hw:0,0"
}

pcm.plug0 {
  type plug
  slave slave0
}
```

|||
```[3,4]
pcm_slave.slave0 {
    pcm "hw:0,0"
    channels 6
    rate 96000
}

pcm.plug0 {
  type plug
  slave slave0
}
```
Here we restrict the device's hardware parameter space
  * Play on 6 channels
  * Set the samplerate to 96kHz

Important:
  * Anything that can be set through the ALSA PCM API can be set in the config file!
  * If you put an unsupported setting in this config, you won't get any errors in your program....
    * Just some errors about samplerate, ... in dmesg

|||
### ALSA mysteries

#### My latency is too high

* Check which card.conf is being loaded   <!-- .element: class="fragment" data-fragment-index="1" -->
  * strace is your friend and wants to help you... unlike ALSA
* Check if some parameters are being set statically    <!-- .element: class="fragment" data-fragment-index="2" -->
  * Period size
  * Buffer size
  * ...

|||
## Some interesting stuff
Exclamation sign causes previous definition to be overridden
```
pcm.!default { type hw card 0 }
```

Can also use:
* ?: assign if not already assigned (so do not override)
* +: create new parameter when necessary
  * Default behaviour and therefore useless
* -: Cause error when trying to assign a parameter which did not previously exist

|||
## More interesting stuff

Functions in the configuration
 * Modify configuration at runtime
   * f.e. through env-variable

```
pcm.!default {
    type plug
    slave.pcm {
        @func getenv
        vars [ ALSAPCM ]
        default "hw:myAwesomeAudioDevice"
    }
}
```

Good luck with that...    <!-- .element: class="fragment" data-fragment-index="1" -->



|||
## More more interesting stuff
Hook functions to configure device when device is opened
* setting volume to 0
* ...

Good luck with that...    <!-- .element: class="fragment" data-fragment-index="1" -->

---
## PCM
Digitized sound
* Samplerate
* Channels
* Sample format

Not as easy as just setting them one by one   <!-- .element: class="fragment" data-fragment-index="1" -->

|||
## PCM configuration

* Constructs a configuration space
* PCM parameters are not independent from each other for some sound cards
  * Cannot combine all sample formats with all sampling rates or channel counts
  * Depends on minimum interrupt 'tick' of HW device
    * Scenario: device supports max X bytes per period to be retrieved
	* Increase the sample size
	* In order to adhere to the X bytes, ALSA decreases the max possible supported samplerate

|||
## PCM configuration

* ALSA constructs an n-dimensional space to limit possible combinations
  * Samplerate dimension
  * Channel dimension
  * Sample format dimension
  * ...
* Therefore, In ALSA API, we set a minimum samplerate
  * snd_pcm_hw_params_set_rate_near(), snd_pcm_hw_params_set_channels_near(), ...
  * This narrows down the possible channels, sample format, ...
  * So order of configuration is important!
    * Order of parameters determines the parameters selected

|||
## PCM configuration

Ok, but how do I know what params are actually set?!

* printf
* /proc/asound/card#/pcm#/sub#/hw_params
* aplay -v

---
## Simple usecase
1 device recording and playing

|||
## Several ways to tackle this

* Synchronous
* Asynchronous
* Polling

In each of these, we can chose two modes:
* Blocking API
* Non-blocking API

|||
#### Synchronous
* one thread per device
  * capture and playback are two separate devices!
* Results in two threads

|||
#### Synchronous blocking

```c
#define AUDIO_SAMPLERATE           16000

#define AUDIO_NR_OF_SAMPLES        128u
#define AUDIO_BYTES_PER_SAMPLE     2u
#define AUDIO_OUTPUT_NR_CHANNELS   2u
#define SAMPLE_BUFFER_SIZE         (AUDIO_NR_OF_SAMPLES * AUDIO_OUTPUT_NR_CHANNELS * AUDIO_BYTES_PER_SAMPLE)

int main(int argc, char **argv)
{
    snd_pcm_t *handle = NULL;
    int period_size = 0;
    int period_time = 0;
    char my_samples[SAMPLE_BUFFER_SIZE];

    snd_pcm_open(&handle, "default:CARD=MyAwesomeAudioCard", SND_PCM_STREAM_PLAYBACK, 0);

    snd_pcm_hw_params_t *hw_params;
    snd_pcm_hw_params_alloca(&hw_params);
    snd_pcm_hw_params_current(handle, hw_params);

    snd_pcm_hw_params_set_access(handle, hw_params, SND_MODE_INTERLEAVED);
    snd_pcm_hw_params_set_format(handle, hw_params, SND_PCM_FORMAT_U16);
    snd_pcm_hw_params_set_rate_near(handle, hw_params, AUDIO_SAMPLERATE);

    snd_pcm_hw_params(handle, hw_params);

    snd_pcm_hw_params_get_period_size(params, period_size, 0);
    snd_pcm_hw_params_get_period_time(params, period_time, NULL);


    snd_pcm_sw_params_t *sw_params = NULL;
    snd_pcm_sw_params_malloc(&sw_params);
    snd_pcm_sw_params_current(handle, sw_params); // AFTER hw params!

    snd_pcm_sw_params_set_start_threshold(handle, sw_params, AUDIO_NR_OF_SAMPLES);

    snd_pcm_sw_params(handle, sw_params);

    // DUMP SETTINGS
    snd_output_t *output = NULL;
    snd_output_stdio_attach(&output, stdout, 0);
    snd_pcm_dump(handle, output);

    // Prepare codec for output
    snd_pcm_drop(handle); // Just to be sure...
    snd_pcm_prepare(handle);

    // ALSA gods demand their buffers to be prefilled with 2 period sizes!
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);

    // Codec is started after this
    snd_pcm_state(handle) == SND_PCM_STATE_RUNNING;

    // Do our thing...
    while(1) {
        int error = snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
        if (error < 0) {
            // xrun recovery
			int recover = snd_pcm_recover(handle, error, 1);
			if (recover < 0) {
			    // all hope is lost....
			}
			// After recovery, need to prefill codec again!
            snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
            snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
        }
    }
}
```

|||
#### Synchronous non-blocking

* Same as blocking
* Functions return new error codes
  * -EBUSY when resource is unavailable
    * aka you did something fishy...
  * -EAGAIN if buffers are full

|||
#### Synchronous non-blocking
```c
#define AUDIO_SAMPLERATE           16000

#define AUDIO_NR_OF_SAMPLES        128u
#define AUDIO_BYTES_PER_SAMPLE     2u
#define AUDIO_OUTPUT_NR_CHANNELS   2u
#define SAMPLE_BUFFER_SIZE         (AUDIO_NR_OF_SAMPLES * AUDIO_OUTPUT_NR_CHANNELS * AUDIO_BYTES_PER_SAMPLE)

int main(int argc, char **argv)
{
    snd_pcm_t *handle = NULL;
    int period_size = 0;
    int period_time = 0;
    char my_samples[SAMPLE_BUFFER_SIZE];

    // Open PCM in non-blocking
    snd_pcm_open(&handle, "default:CARD=MyAwesomeAudioCard", SND_PCM_STREAM_PLAYBACK, SND_PCM_NONBLOCK);

    // setup rest of stuff...

    // Prepare codec for output
    snd_pcm_drop(handle); // Just to be sure...
    snd_pcm_prepare(handle);

    // ALSA gods demand their buffers to be prefilled with 2 period sizes!
	snd_pcm_non_block(handle, 0); // We want to block for the prefill
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
	snd_pcm_non_block(handle, 1); // get back to non-blocking

    // Codec is started after this
    snd_pcm_state(handle) == SND_PCM_STATE_RUNNING;

    // Do our thing...
    while(1) {
        int error = snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
		if (error == -EAGAIN) {
		    sleep(100);
			continue;
		}
		// xrun recovery stuff
    }
}
```

|||
#### Asynchronous
  * 'Microcontroller'-way (patent pending)
  * Seems natural/elegant
    * Why let us do the waiting when ALSA can just inform us when it wants samples?
  * Uses SIGIO by default
    * Wizards can redefine it to a realtime signal (actual quote from ALSA docs...)
  * ALSA Driver has to raise the signal
    * So it's not supported by a lot of drivers
    * Can conflict with other modules/libraries/... using the same signal
	  * Hence, you have to be a wizard to get it working

|||
#### Asynchronous
* Not very stable...
  * Seems confirmed by the fact that nobody seems to use it
  * Big players (PulseAudio/JACK/PortAudio) don't use it
* My official recommendation: don't use it if you want to maintain your sanity...

|||
#### Asynchronous
```c
#define AUDIO_SAMPLERATE           16000

#define AUDIO_NR_OF_SAMPLES        128u
#define AUDIO_BYTES_PER_SAMPLE     2u
#define AUDIO_OUTPUT_NR_CHANNELS   2u
#define SAMPLE_BUFFER_SIZE         (AUDIO_NR_OF_SAMPLES * AUDIO_OUTPUT_NR_CHANNELS * AUDIO_BYTES_PER_SAMPLE)

static void my_signal_callback(snd_async_handler_t *ahandler)
{
    snd_pcm_t *pcm_handle = snd_async_handler_get_pcm(ahandler);
    void *my_priv_data = snd_async_hanlder_get_callback_private(ahandler);
    char my_samples[SAMPLE_BUFFER_SIZE] = { 0 };

    snd_pcm_sframes_t avail = 0;

    avail = snd_pcm_avail_update(pcm_handle);

    if (avail < 0) {
        // xrun recovery
    }

    while (avail >= AUDIO_NR_SAMPLES) {
        snd_pcm_writei(pcm_handle, my_samples, AUDIO_NR_SAMPLES);
        avail = snd_pcm_avail_update(pcm_handle);
		// xrun checking...
    }
}

int main(int argc, char **argv)
{
    snd_pcm_t *handle = NULL;
    snd_async_handler_t *async_handle = NULL;
    void *my_priv_data = NULL;
    int period_size = 0;
    int period_time = 0;
    char my_samples[SAMPLE_BUFFER_SIZE];

    // Open pcm, non-blocking or blocking, your choice
    snd_pcm_open(&handle, "default:CARD=MyAwesomeAudioCard", SND_PCM_STREAM_PLAYBACK, 0);
    // Just... for the love of god.... don't use SND_PCM_ASYNC!!!
    // snd_pcm_open(&handle, "default:CARD=MyAwesomeAudioCard", SND_PCM_STREAM_PLAYBACK, SND_PCM_ASYNC);

    // setup rest of stuff...

    // Install callback
    snd_pcm_add_pcm_handler(&async_handle, handle, my_signal_callback, my_priv_data);

    // Prepare codec for output
    snd_pcm_drop(handle); // Just to be sure...
    snd_pcm_prepare(handle);

    // ALSA gods demand their buffers to be prefilled with 2 period sizes!
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);
    snd_pcm_writei(handle, my_samples, AUDIO_NR_OF_SAMPLES);

    // Codec is started after this
    snd_pcm_state(handle) == SND_PCM_STATE_RUNNING;

    while(1) {
        // everything is handled in callback
        sleep(100);
    }
}
```

|||
#### Polling
  * Closest to the async model and still managable/stable
  * We ask ALSA for file descriptors of the audio device we can poll
    * Depending on Playback/Recording, poll returns POLLOUT/POLLIN
  * Sweet!
   * No need for sleeps, waiting, ....

|||
#### Polling
Oh you naive fool....
  * ALSA can generate 'NULL' events
    * Yes, stated by their official documentation...
  * So your thread is always waking up and checking for null events
  * You still have to throttle the thread/poll yourself....

|||
#### Polling

```c
#include <stdio.h>
#include <alsa/asoundlib.h>

#define AUDIO_SAMPLERATE           16000

#define AUDIO_NR_OF_SAMPLES        128u
#define AUDIO_BYTES_PER_SAMPLE     2u
#define AUDIO_OUTPUT_NR_CHANNELS   2u
#define SAMPLE_BUFFER_SIZE         (AUDIO_NR_OF_SAMPLES * AUDIO_OUTPUT_NR_CHANNELS * AUDIO_BYTES_PER_SAMPLE)

#define PERIOD_SIZE_US             8000u

int main(int argc, char **argv)
{
    int error = 0;
    struct pollfd *ufds = NULL;
    snd_pcm_t *pcm_handle = NULL;
    int poll_fd_count = 0;
    char my_samples[SAMPLE_BUFFER_SIZE] = { 0 };
    int revents = 0;

    snd_pcm_open(&pcm_handle, "default:CARD=MyAwesomeAudioCard", SND_PCM_STREAM_PLAYBACK, 0);

	// default setup stuff here....

    // Set start value
    if (err < snd_pcm_sw_params_set_avail_min(handle, swparams, period_size)) {
        printf("Unable to set avail min for playback: %s\n", snd_strerror(err));
        return err;
    }

    // Enable poll event
    err = snd_pcm_sw_params_set_period_event(handle, swparams, 1);
    if (err < 0) {
        printf("Unable to set period event: %s\n", snd_strerror(err));
        return err;
    }

    // Set SW params
    if ((err = snd_pcm_sw_params(handle, swparams)) < 0) {
        printf("ERROR: Can't set software parameters. %s\n", snd_strerror(err));
        return err;
    }

    // Get number of file descriptors (can be more than 1, depends on driver)
    poll_fd_count = snd_pcm_poll_descriptors_count(pcm_handle);

    // Get the actual file descriptors
    ufds = malloc(sizeof(*ufds) * poll_fd_count);
    snd_pcm_poll_descriptors(pcm_handle, ufds, poll_fd_count);

    // start device
    snd_pcm_start(pcm_handle);

    while (1) {
        poll(ufds, poll_fd_count, -1);
        snd_pcm_poll_descriptors_revents(pcm_handle, ufds, poll_fd_count, &revents);
        if (revents & POLLERR) {
            // All hope is lost...
        } else if (revents & POLLOUT) {
            error = snd_pcm_writei(pcm_handle, my_samples, AUDIO_NR_OF_SAMPLES);
            if (error < 0) {
                // xrun recovery
            }
        }

		// Throttle our thread...
		usleep(PERIOD_SIZE_US / 2)
    }

    return 0;
}
```

|||
#### ALSA quirks
* alsa-lib startup behaviour for output devices
  * Need to be prefilled with 2 * period_size
  * Just because
  * If not... xrun after a while...without them being reported in your app
    * Errors in dmesg
* Maintain handshake with library
  * Get state from pcm handle
  * Depending on that state, do stuff
    * recover from xrun
	* prepare again after xrun
	* prefill before writing your own audio!
  * https://www.alsa-project.org/alsa-doc/alsa-lib/pcm.html
* Input/output settings linked to each other on same device
  * Sounds logical...
  * ...However, ALSA will allow to let you define f.e. 48kHz on input and 16kHz on output
  * Even though this isn't possible due to HW constraints (it's the same device!)

---
## Complex usecase
Several devices

|||
## Blocking
Extrapolate the blocking usecase
* 1 device == 1 thread
* Simple
* But resource heavy....
  * f.e. 8 Audio devices, full duplex (playback and recording)
  * 8 devices x 2 threads = 16!

|||
## Polling
* Bookkeeping
  * Figuring out which device generated event
* One device can influence others
  * If one device takes too long writing/reading,
    the others can experience xrun
* But, this seems to be the only stable/flexible way to handle multiple alsa devices in a performance/resource-friendly way.

|||
#### Polling

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>
#include <pthread.h>

#include <alsa/asoundlib.h>
#include <alloca.h>

#define AUDIO_NR_OF_SAMPLES     128u
#define AUDIO_BYTES_PER_SAMPLE  2u
#define AUDIO_NR_OF_CHANNELS    2u
#define SAMPLE_BUFFER_SIZE      (AUDIO_NR_OF_SAMPLES * AUDIO_NR_OF_CHANNELS * AUDIO_BYTES_PER_SAMPLE)

static char *device_names[] = {
	"sysdefault:CARD=AudioBoard00",
	"sysdefault:CARD=AudioBoard01",
	"sysdefault:CARD=AudioBoard02",
	"sysdefault:CARD=AudioBoard03",
	"sysdefault:CARD=AudioBoard04",
	"sysdefault:CARD=AudioBoard05",
	"sysdefault:CARD=AudioBoard06",
	"sysdefault:CARD=AudioBoard07",
};

#define NR_OF_DEVICES (sizeof(device_names)/sizeof(*device_names))

static void *read_thread_func(void *cookie)
{
	int error = 0;

    struct pollfd *ufds = NULL;
	snd_pcm_t *pcm_handles[NR_OF_DEVICES];
	int poll_fd_count[NR_OF_DEVICES] = { 0 };
	int total_poll_fd_count = 0;
	int poll_fd_count_offset[NR_OF_DEVICES] = { 0 };

	for (int i = 0; i< NR_OF_DEVICES; ++i) {
		if (snd_pcm_open(&pcm_handles[i], device_names[i], SND_PCM_STREAM_CAPTURE, 0) < 0) {
			printf("Failed to open device '%s'\n", device_names[i]);
			goto _free_resources;
		}

        // setup hw/sw params of alsa device...

		poll_fd_count[i] = snd_pcm_poll_descriptors_count(pcm_handles[i]);
		if (poll_fd_count[i] <= 0) {
			printf("Invalid poll descriptors count for device '%s'\n", device_names[i]);
			goto _free_resources;
		}

		total_poll_fd_count += poll_fd_count[i];
	}

	ufds = malloc(sizeof(*ufds) * total_poll_fd_count);
	if (ufds == NULL) {
		printf("Not enough memory\n");
		goto _free_resources;
	}
	memset(ufds, 0, sizeof(*ufds) * total_poll_fd_count);

    for (int i = 1; i < NR_OF_DEVICES; ++i) {
	    for (int j = 0; j < i; j++) {
		    poll_fd_count_offset[i] += poll_fd_count[j];
		}
    }

	/* Get poll file descriptors */
    for (int i = 0; i< NR_OF_DEVICES; ++i) {
    	if ((error = snd_pcm_poll_descriptors(pcm_handles[i], ufds + poll_fd_count_offset[i], poll_fd_count[i])) < 0) {
			printf("Unable to obtain poll descriptors for playback: %s\n", snd_strerror(error));
			goto _free_resources;
		}
	}

	/* Start readers */
	for (int i = 0; i < NR_OF_DEVICES; i++) {
        if (snd_pcm_start(pcm_handles[i]) < 0) {
            printf("Failed to start pcm '%s'", device_names[i]);
            goto _free_resources;
        }
    }

    char black_hole[SAMPLE_BUFFER_SIZE] = {0};

	while (1) {
		unsigned short revents = 0;

		poll(ufds, total_poll_fd_count, -1);

		for (int i = 0; i < NR_OF_DEVICES; i++) {
			revents = 0;
			snd_pcm_poll_descriptors_revents(pcm_handles[i], ufds + poll_fd_count_offset[i], poll_fd_count[i], &revents);
			if (revents & POLLERR) {
				printf("ERROR:Failed to poll device %s\n", device_names[i]);
				break;
			} else if (revents & POLLIN) {
				error = snd_pcm_readi(pcm_handles[i], black_hole, AUDIO_NR_OF_SAMPLES);
				if (error < 0) {
				    // xrun recovery
				}
			}
		}
	}

	printf("Reader done...\n");

	for (int i = 0; i < NR_OF_DEVICES; ++i) {
		snd_pcm_drop(pcm_handles[i]);
		snd_pcm_close(pcm_handles[i]);
	}

_free_resources:
    // clean up your shit
}

static void *write_thread_func(void *cookie)
{
	int error = 0;
    struct pollfd *ufds = NULL;
	snd_pcm_t *pcm_handles[NR_OF_DEVICES];
	int poll_fd_count[NR_OF_DEVICES] = { 0 };
	int total_poll_fd_count = 0;
	int poll_fd_count_offset[NR_OF_DEVICES] = { 0 };

	for (int i = 0; i< NR_OF_DEVICES; ++i) {
		if (snd_pcm_open(&pcm_handles[i], device_names[i], SND_PCM_STREAM_PLAYBACK, 0) < 0) {
			printf("Failed to open device '%s'\n", device_names[i]);
			goto _free_resources;
		}

		// setup hw/sw params

		poll_fd_count[i] = snd_pcm_poll_descriptors_count(pcm_handles[i]);
		if (poll_fd_count[i] <= 0) {
			printf("Invalid poll descriptors count for device '%s'\n", device_names[i]);
			goto _free_resources;
		}

		total_poll_fd_count += poll_fd_count[i];
	}

	ufds = malloc(sizeof(*ufds) * total_poll_fd_count);
	if (ufds == NULL) {
		printf("Not enough memory\n");
		goto _free_resources;
	}
	memset(ufds, 0, sizeof(*ufds) * total_poll_fd_count);

    for (int i = 1; i < NR_OF_DEVICES; ++i) {
		for (int j = 0; j < i; j++) {
			poll_fd_count_offset[i] += poll_fd_count[j];
        }
    }

	/* Get poll file descriptors */
    for (int i = 0; i< NR_OF_DEVICES; ++i) {
		if ((error = snd_pcm_poll_descriptors(pcm_handles[i], ufds + poll_fd_count_offset[i], poll_fd_count[i])) < 0) {
			printf("Unable to obtain poll descriptors for playback: %s\n", snd_strerror(error));
			goto _free_resources;
		}
	}

	/* Prefill */
	for (int i = 0; i < NR_OF_DEVICES; i++) {
		char mybuff[SAMPLE_BUFFER_SIZE * 2] = {0 };
		if (snd_pcm_writei(pcm_handles[i], mybuff, AUDIO_NR_OF_SAMPLES * 2) < 0 ) {
			printf("could not prefill\n");
			goto _free_resources;
		}
	}

	while (true) {
		unsigned short revents = 0;

		poll(ufds, total_poll_fd_count, -1);

		for (int i = 0; i < NR_OF_DEVICES; i++) {
			revents = 0;
			snd_pcm_poll_descriptors_revents(pcm_handles[i], ufds + poll_fd_count_offset[i], poll_fd_count[i], &revents);
			if (revents & POLLERR) {
				printf("ERROR:Failed to poll device '%s'\n", device_names[i]);

				if (snd_pcm_state(pcm_handles[i]) == SND_PCM_STATE_XRUN ||
				    snd_pcm_state(pcm_handles[i]) == SND_PCM_STATE_SUSPENDED) {
					error = snd_pcm_state(pcm_handles[i]) == SND_PCM_STATE_XRUN ? -EPIPE : -ESTRPIPE;
					if (xrun_recovery(pcm_handles[i], error) < 0) {
						printf("Write error: %s\n", snd_strerror(error));
						exit(EXIT_FAILURE);
					}
				} else {
					LOG_ERROR("Wait for poll failed '%s'\n", device_names[i]);
					break;
				}


				/* _is_writer_done = true; */
				continue;
			} else if (revents & POLLOUT) {
				char *ptr = cbuffer_get_read_pointer(_loopback_buffers[i]);
				if (!ptr) {
					continue;
				}
				error = snd_pcm_writei(pcm_handles[i], ptr, AUDIO_NR_OF_SAMPLES);
				if (error < 0) {
					if (xrun_recovery(pcm_handles[i], error) < 0) {
						printf("Write error: %s\n", snd_strerror(error));
						exit(EXIT_FAILURE);
					}
					break;
				} else {
					/* printf("%d: Written %d samples in device %s\n", ++count, error, device_names[i]); */
				}
				cbuffer_signal_element_read(_loopback_buffers[i]);
			}
		}
	}

	printf("writer done...\n");

	for (int i = 0; i < NR_OF_DEVICES; ++i) {
		snd_pcm_drain(pcm_handles[i]);
		snd_pcm_close(pcm_handles[i]);
	}

	printf("free resources writer...\n");
_free_resources:
    // clean up your shit

	return NULL;
}

int main(int argc, char **argv)
{
	pthread_t write_thread;
	pthread_t read_thread;
	int error = 0;

	printf("Start playback/record with following devices:\n");
	for (int i = 0; i < NR_OF_DEVICES; ++i) {
		printf("%s\n", device_names[i]);
	}

    if (pthread_create(&read_thread, NULL, read_thread_func, NULL) < 0) {
		printf("Failed to create reader thread");
		goto _free_resources;
    }

    if (pthread_create(&write_thread, NULL, write_thread_func, NULL) < 0) {
        printf("Failed to create reader thread");
		goto _free_resources;
    }

	pthread_join(write_thread, NULL);
	pthread_join(read_thread, NULL);

	return 0;
}
```

---
## Tips and tricks
* Thread safety
  * Opening audio devices is thread safe and returns a handle
  * User is responsible for serializing acces to handle-related functions
    * So you have to provide your own locking scheme if handle is shared!
  * Standalone (non-handle) functions are thread safe though
* Disable multi-thread support at runtime
  * LIBASOUND_THREAD_SAFE=0
* Check the official examples
  * The API doesn't work as you would think it works
* Check alsa-utils

|||
## Tips and tricks
* Check alsa-mixer for volume control implementation
  * Human ear is an a-hole
* Check PortAudio/PulseAudio/JACK/...
  * Not just for 'code ideas'
  * You can use them iso ALSA
    * But check performance impact
* Useful tools
  * aplay
    * list all devices: aplay -l
    * list configuration of device: aplay -v
* Some functions are notorious for not being stable
  * snd_pcm_drain
	* Comment in PortAudio:src/hostapi/alsa/pa_linux_alsa.c:AlsaStop: "snd_pcm_drain can hang forever."
	* Commit from 2013...

|||
## Tips and tricks
* ALSA output devices startup behaviour
  * Need to be prefilled with 2 * period_size
  * Just because
  * If not... xrun after a while
* Configured audio devices
  * /proc/asound/cards
* State of sound devices stored persistently
  * Alsa-utils
  * /var/lib/alsa/asound.state
  * Can be disabled


|||
## Future stuff
ALSA/alsa-lib introduced usecase managers
* Headset
* Speakers
* Microphone
* ...

Used in Ubuntu Touch (Nexus 7)
* Ability to set actions
  * EnableSpeakers
  * DisableMicrophone


---
## Sources
* Wiki https://alsa-project.org/wiki
  * Contains a lot of dead links
  * Mostly replaced with kernel info (see below)
* Unofficial wiki https://alsa.opensrc.org/
  * Is more honest about stuff that is broken etc.
* Kernel info https://www.kernel.org/doc/html/v4.17/sound/index.html
  * Writing an ALSA driver https://www.kernel.org/doc/html/v4.17/sound/kernel-api/writing-an-alsa-driver.html


|||
## Sources
* Some guy that also struggled http://www.volkerschatz.com/noise/alsa.html
* Sample programs http://equalarea.com/paul/alsa-audio.html
* alsa-lib doxygen https://www.alsa-project.org/alsa-doc/alsa-lib/
  * Handshake between app and lib https://www.alsa-project.org/alsa-doc/alsa-lib/pcm.html
* Arch is the best wiki ever https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture
* Recent ALSA examples https://github.com/OpenPixelSystems/c-alsa-examples
