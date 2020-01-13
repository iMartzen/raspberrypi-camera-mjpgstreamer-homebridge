# Raspberrypi-camera-mjpegstreamer-homebridge
This a tutorial about how to setup a Raspberry Pi Zero W to display in Homekit. This tutorial is not complete. It doesn't show how to set up homebridge in example.

# What do I need? 

- [Raspberry Pi Zero W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/)
- SD card (with a minimum of 4GB) 
- Raspberry Pi Zero W Adapter
- Raspberry Pi Zero W Camera Cable 
- Raspberry Pi Camera Board v2 
- Raspberry Pi Camera Enclosure (i.e. You can 3D print [Raspberry Pi Enclosure by @brutella from the hkcam project](https://github.com/brutella/hkcam/tree/master/enclosure ))

# Let's get started! 
## Flash Raspbian on SD card
- Download the latest Raspbian Lite image from: [https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/)
- Download and install the latest Etcher flash tool from: [https://www.balena.io/etcher/](https://www.balena.io/etcher/)
- Flash Raspbian Lite with Etcher

## Enable ssh
- Create an empty ssh file on the root of the sd card
```bash
$ touch ssh
```

## Give direct WIFI access
- Create an file called: wpa_supplicant.conf with content as below, change SSID name + PASSWORD 
```bash
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
ssid="<<SSID>>"
psk="<<PASSWORD>>"
}
```

## Boot the RPI with the SD card inserted
- Detect it's ip address (default assigned by dhcp)
```bash
$ nmap -sn 10.0.100.1-100
```
## Login with ssh (default password: raspberry)
```bash
ssh pi@<pi_ip_address>
```

## Changing your username 
- Set passed for root 
```bash
$ sudo passwd root 
```
- Make it possible to login via ssh for root 
```bash
$ sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
```
- Restart ssh
```bash
$ sudo /etc/init.d/ssh reload 
```
- Login as root
```bash
$ ssh root@<ipadres>
```
- Change username and make home directory
```bash
$ usermod -l <username> -d /home/<username> -m pi 
```
- Change name in the group to username 
```bash
$ groupmod -n <username> pi 
```

## SSH configuration
- Generate new SSH Key (anser defaults with enter)											
```bash
$ ssh-keygen
```
- Add your public keys to .ssh/authorized_keys file 
```bash
$ echo '<public key>' > ~/.ssh/authorized_keys 
```
- Test your key, exit the connection and log back in. There should be no prompt for password! 

- Disable root login and password based login
```bash
$ sudo sed -i 's/PermitRootLogin yes/#PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
$ sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no' /etc/ssh/sshd_config
$ sudo sed -i 's/UsePAM yes/usePAM no/' /etc/ssh/sshd_config
$ sudo sed -i 's/PrintMotd no/PrintMotd yes/' /etc/ssh/sshd_config 
$ sudo /etc/init.d/ssh reload
```

## Update and upgrade the lastest packages & update Raspberry PI firmware
```bash
$ sudo apt-get update && sudo apt-get -y upgrade && sudo apt auto-clean && sudo rpi-update
```

## Reboot is needed
```bash
sudo reboot now 
```

## Change Raspbian config
```bash							
$ sudo raspi-config
    $ Network Configuration
        - change hostname (i.e camera-1)
        - WIFI SSID + WW 
    $ Localisation options:
        - change locale: en_GB.UTF-8 UTF-8 and en_US.UTF-8 UTF-8
        - locale default: none
        - change timezone: Europe/Amsterdam
    $ Interfacing options: 
        - enable camera: yes
```

## Reboot is needed
```bash
sudo reboot now 
```

## Check if configuration works, making a photo
```bash
$ cd ~ 
$ raspistill -o cam.jpg
```
- On your **local** computer: 
```bash
	$ scp pi@raspberry.local:~/cam.jpg 
```
- Check if the image contains the image you expect, where the camera is pointed at. 

## Prepare the system for the software mjpg-streamer
```bash
$ sudo apt-get install cmake libjpeg8-dev git gcc g++ 
```

## Clone repository & compile the software 
```bash
$ cd ~
$ git clone https://github.com/jacksonliam/mjpg-streamer.git
$ cd mjpg-streamer/mjpg-streamer-experimental
$ make
$ sudo make install 
```

# Test the installed software
```bash
$ export LD_LIBRARY_PATH=.
$ ./mjpg_streamer -o "output_http.so -w ./www" -i "input_raspicam.so"
```

## Browse to
- http://<<ip or dns>>:8080
- If you see the MJPG Streamer welcome page, it works! 

# Enable it on reboot
```bash
$ nano /home/<username>/mjpg-streamer.sh
```
- Copy this text in the mjpg-streamer.sh file 
```bash 
#!/bin/bash
# chmod +x mjpg-streamer.sh
# Crontab: @reboot /home/pi/mjpg-streamer/mjpg-streamer.sh start
# Crontab: @reboot /home/pi/mjpg-streamer/mjpg-streamer-experimental/mjpg-streamer.sh start

MJPG_STREAMER_BIN="/usr/local/bin/mjpg_streamer"  # "$(dirname $0)/mjpg_streamer"
MJPG_STREAMER_WWW="/usr/local/share/mjpg-streamer/www"
MJPG_STREAMER_LOG_FILE="${0%.*}.log"  # "$(dirname $0)/mjpg-streamer.log"
RUNNING_CHECK_INTERVAL="2" # how often to check to make sure the server is running (in seconds)
HANGING_CHECK_INTERVAL="3" # how often to check to make sure the server is not hanging (in seconds)

VIDEO_DEV="/dev/video0"
FRAME_RATE="5"
QUALITY="80"
RESOLUTION="1280x720"  # 160x120 176x144 320x240 352x288 424x240 432x240 640x360 640x480 800x448 800x600 960x544 1280x720 1920x1080 (QVGA, VGA, SVGA, WXGA)   #  lsusb -s 001:006 -v | egrep "Width|Height" # https://www.textfixer.com/tools/alphabetical-order.php  # v4l2-ctl --list-formats-ext  # Show Supported Video Formates
PORT="8080"
YUV="yes"

################INPUT_OPTIONS="-r ${RESOLUTION} -d ${VIDEO_DEV} -f ${FRAME_RATE} -q ${QUALITY} -pl 60hz"
INPUT_OPTIONS="-r ${RESOLUTION} -d ${VIDEO_DEV} -q ${QUALITY} -pl 60hz --every_frame 2"  # Limit Framerate with  "--every_frame ", ( mjpg_streamer --input "input_uvc.so --help" )


if [ "${YUV}" == "true" ]; then
	INPUT_OPTIONS+=" -y"
fi

OUTPUT_OPTIONS="-p ${PORT} -w ${MJPG_STREAMER_WWW}"

# ==========================================================
function running() {
    if ps aux | grep ${MJPG_STREAMER_BIN} | grep ${VIDEO_DEV} >/dev/null 2>&1; then
        return 0

    else
        return 1

    fi
}

function start() {
    if running; then
        echo "[$VIDEO_DEV] already started"
        return 1
    fi

    export LD_LIBRARY_PATH="$(dirname $MJPG_STREAMER_BIN):."

	echo "Starting: [$VIDEO_DEV] ${MJPG_STREAMER_BIN} -i \"input_uvc.so ${INPUT_OPTIONS}\" -o \"output_http.so ${OUTPUT_OPTIONS}\""
    ${MJPG_STREAMER_BIN} -i "input_uvc.so ${INPUT_OPTIONS}" -o "output_http.so ${OUTPUT_OPTIONS}" >> ${MJPG_STREAMER_LOG_FILE} 2>&1 &

    sleep 1

    if running; then
        if [ "$1" != "nocheck" ]; then
            check_running & > /dev/null 2>&1 # start the running checking task
            check_hanging & > /dev/null 2>&1 # start the hanging checking task
        fi

        echo "[$VIDEO_DEV] started"
        return 0

    else
        echo "[$VIDEO_DEV] failed to start"
        return 1

    fi
}

function stop() {
    if ! running; then
        echo "[$VIDEO_DEV] not running"
        return 1
    fi

    own_pid=$$

    if [ "$1" != "nocheck" ]; then
        # stop the script running check task
        ps aux | grep $0 | grep start | tr -s ' ' | cut -d ' ' -f 2 | grep -v ${own_pid} | xargs -r kill
        sleep 0.5
    fi

    # stop the server
    ps aux | grep ${MJPG_STREAMER_BIN} | grep ${VIDEO_DEV} | tr -s ' ' | cut -d ' ' -f 2 | grep -v ${own_pid} | xargs -r kill

    echo "[$VIDEO_DEV] stopped"
    return 0
}

function check_running() {
    echo "[$VIDEO_DEV] starting running check task" >> ${MJPG_STREAMER_LOG_FILE}

    while true; do
        sleep ${RUNNING_CHECK_INTERVAL}

        if ! running; then
            echo "[$VIDEO_DEV] server stopped, starting" >> ${MJPG_STREAMER_LOG_FILE}
            start nocheck
        fi
    done
}

function check_hanging() {
    echo "[$VIDEO_DEV] starting hanging check task" >> ${MJPG_STREAMER_LOG_FILE}

    while true; do
        sleep ${HANGING_CHECK_INTERVAL}

        # treat the "error grabbing frames" case
        if tail -n2 ${MJPG_STREAMER_LOG_FILE} | grep -i "error grabbing frames" > /dev/null; then
            echo "[$VIDEO_DEV] server is hanging, killing" >> ${MJPG_STREAMER_LOG_FILE}
            stop nocheck
        fi
    done
}

function help() {
    echo "Usage: $0 [start|stop|restart|status]"
    return 0
}

if [ "$1" == "start" ]; then
    start && exit 0 || exit -1

elif [ "$1" == "stop" ]; then
    stop && exit 0 || exit -1

elif [ "$1" == "restart" ]; then
    stop && sleep 1
    start && exit 0 || exit -1

elif [ "$1" == "status" ]; then
    if running; then
        echo "[$VIDEO_DEV] running"
        exit 0

    else
        echo "[$VIDEO_DEV] stopped"
        exit 1

    fi

else
    help

fi
```


- Make the file executable 
```bash
$ chmod +x mjpg-streamer.sh 
```
- Edit conjob configuration 
```bash
$ crontab -e 
```
- Add line for reboot 
```bash
$ @reboot /home/pi/mjpg-streamer.sh start
```

# Add camera to homebridge: 
- Ensure that homebridge included the package: ffmpeg
    - To do this included in stack/compose file in the environment section: PACKAGES=ffmpeg (https://github.com/oznu/docker-homebridge#parameters)
- Install plugin homebridge-camera-ffmpeg
    - Click on 'plugin'
    - Search for homebridge-camera-ffmpeg
    - Click on ‘install’ 
- Add to homebridge config: 

```json
"platforms": [
    {
        "name": "Camera ffmpeg",
        "cameras": [
            {
                "name": "Default Camera or <other name>" ,
                "videoConfig": {
                    "source": "-re -f mjpeg -i http://<ip-or-dns-of-your-camera>:8080/?action=stream",
                    "stillImageSource": "-f mjpeg -i http://<ip-or-dns-of-your-camera>:8080/?action=snapshot",
                    "maxStreams": 4,
                    "maxWidth": 640,
                    "maxHeight": 480,
                    "maxFPS": 10,
                    "debug": true
                }
            }
        ],
        "platform": "Camera-ffmpeg"
    }   
],
```

# Add to Homekit: 
- Click on ‘+’ 
- Add accessory
- Enter pin manually  
- Pin is known in your homebridge instance 

# Sources used for this: 
- https://www.raspberrypi.org/documentation/installation/sd-cards.md
- https://www.raspberrypi.org/documentation/usage/camera/README.md
- https://www.raspberrypi.org/documentation/usage/camera/raspicam/raspistill.md 
- https://www.raspberrypi.org/documentation/hardware/camera/ 
- https://github.com/KhaosT/homebridge-camera-ffmpeg/wiki/Raspberry-PI
- https://github.com/jacksonliam/mjpg-streamer
- https://github.com/oznu/docker-homebridge#parameters  
- https://github.com/cncjs/cncjs/wiki/Setup-Guide:-Raspberry-Pi-%7C-MJPEG-Streamer-Install-&-Setup-&-FFMpeg-Recording 