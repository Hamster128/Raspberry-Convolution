# Raspberry-Convolution

Recently i stumbled over this YouTube video which shows how to measure your listening room and make an impulse response file with "REW" to use it with Equilizer APO on a PC for real time room correction.

https://youtu.be/5YcH7j2-L1Y

REW (Room EQ Wizard)
https://www.roomeqwizard.com

Equalizer APO
https://sourceforge.net/projects/equalizerapo


It worked that great that i wanted the correction also for my TV which is connected to the amp via digital optical cable. The idea was to use a Raspberry Pi with the Hifiberry DIGI+ board to plug it between the TV and the amp and let it run the convolution filter in real time.
After trying around a few days i came up with the following solution:


## Preparing Rapsberry Pi

With "Raspberry Pi Manager" i installed Raspberry Pi OS Lite 64 (kernel 5.15.84-v8+) including the WiFi settings on the SD card.


## Activate HifiBerry DIGI+

    sudo nano /boot/config.txt

add the follwing lines to load the DIGI+ driver:

    # load HifiBerry DIGI+
    force_eeprom_read=0
    dtoverlay=hifiberry-digi

list sound cards:

    cat /proc/asound/cards


## Configure alsa:

    sudo nano /etc/asound.conf

add the following lines with the number of your card

    pcm.hifiberry {
      type hw card 1  
    }
    ctl.hifiberry {
      type hw card 1
    }

list alsa devices:

    aplay -L

list record devices:

    arecord -l

play music:

    flac -c -d hello.flac | aplay -D hifiberry


## Install bmc0 dsp:
https://github.com/bmc0/dsp

download and unzip source:

    wget https://github.com/bmc0/dsp/archive/refs/tags/v1.9.zip
    unzip v1.9.zip
    rm *.zip

install dependencies:

    sudo apt-get install -y libtool libzita-convolver-dev libasound2-plugins libasound2-dev libfftw3-dev 
    sudo apt-get install -y libsndfile-dev ladspa-sdk libzita-alsa-pcmi0 libzita-alsa-pcmi-dev alsa-tools zita-alsa-pcmi-utils

configure and make:

    cd dsp*
    ./configure
    make
    sudo make install

test dsp:

    dsp hello.flac -o hifiberry highpass 1.8k 0.5

Test convolver (ir44.wav is the impulse response file exported form REW):

    dsp hello.flac -o hifiberry -e s24 zita_convolver ir44.wav

Loopback with dsp:

    dsp -c 2 -e s16 -r 48000 -t alsa hifiberry -o hifiberry zita_convolver ir48.wav

Start at boot:

    sudo nano /etc/systemd/system/dsp.service

content for new systemd config file:

    [Unit]
    Description=dsp for living room speakers

    # don`t ever stop restarting this service
    StartLimitIntervalSec=0

    [Service]
    WorkingDirectory=/home/henry

    # check if there is some sound incoming, othersie don`t start dsp to prevent any unwanted output
    ExecStartPre=arecord -f S24_LE -c 2 -r 48000 -D hifiberry -s 1

    # start the dsp as a loop back device
    ExecStart=dsp -c 2 -e s16 -r 48000 -t alsa hifiberry -o hifiberry zita_convolver ir48.wav

    # restart this service in any case, even on exit 0 success
    Restart=always

    # restart this service after 3 seconds
    RestartSec=3s

    [Install]
    WantedBy=multi-user.target

Install new service:

    sudo systemctl enable dsp

Start new service:

    sudo systemctl start dsp

Keep process priority high:

    sudo crontab -e

add this line:

    * * * * * ls /proc/$(pidof dsp)/task | xargs sudo renice -19

after making changes to /etc/systemd/system/dsp.service:

    sudo systemctl daemon-reload
    sudo systemctl restart dsp

if had to shorten the impulse response wave file a lot otherwise the latency was too long. The zita_convolver should have a low latency even with longer impulse response files, but that didn`t work out that way. So i kept only about 100ms of the original wave file. (used Audacity)


## Install dsp as alsa audio device

this makes it possible to play audio with other Raspberry Pi software through the dsp

create the dsp configuration file

    sudo mkdir /etc/ladspa_dsp 
    sudo nano /etc/ladspa_dsp/config_living_room_speakers

add these lines:

    input_channels=2
    output_channels=2
    LC_NUMERIC=C
    effects_chain=zita_convolver /etc/ladspa_dsp/ir48.wav

copy impulse response file to the configuration path:

    sudo cp ir48.wav /etc/ladspa_dsp

setup the alsa configuration:

    sudo nano /etc/asound.conf

add the following:

    # output to "dsp" for convolved output to hifiberry
    # first use type plug to convert to format FLOAT (ladspa can only process FLOAT)
    pcm.dsp {
      type plug
      slave {
        format FLOAT
        pcm convolver
      }
      hint {
        show on
        description "Living Room Speakers Convolver."
      }
    }

    # the actual convolver
    pcm.convolver {
      type ladspa
      channels 2
      path "/usr/lib/ladspa"
      playback_plugins [{
        label "ladspa_dsp:living_room_speakers"
      }]
      slave.pcm digital_out
    }

    # use plug to convert FLOAT back to S24_LE (HifiBerry DIGI+ can only output S16_LE or S24_LE)
    pcm.digital_out {
      type plug
      slave {
        format S24_LE
        pcm hifiberry
      }
    }

Test own audio device:

    flac -c -d hello.flac | aplay -D dsp
