# Verwendete Anleitungen
https://medium.com/@hungryspider/building-custom-linux-for-raspberry-pi-using-buildroot-f81efc7aa817  
https://kickstartembedded.com/2021/12/19/yocto-part-2-setting-up-ubuntu-host/  
https://kickstartembedded.com/2021/12/22/yocto-part-4-building-a-basic-image-for-raspberry-pi/  
https://www.codeinsideout.com/blog/yocto/raspberry-pi/#run-on-hardware  


# Benötigte Packete auf Host
In den meisten Anleitung wird noch das Packet 'pylint3' mit aufgeführt, dieses  
ist jedoch deprecated und wird nicht mehr benötigt.
```
sudo apt install gawk wget git git-lfs tar diffstat unzip texinfo gcc
build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils
debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev
xterm python3-subunit mesa-common-dev zstd liblz4-tool
```

balenaEtcher fürs Flashen des Images auf die SD Karte:
```
wget https://github.com/balena-io/etcher/releases/download/v1.18.11/balena-etcher_1.18.11_amd64.deb
sudo dpkg -i balena-etcher_1.18.11_amd64.deb
```

# Build System - Buildroot
## Buildroot Repo clonen
Die aktuelle LTS Version ist die 2024.02, diese wie folgt herunterladen
```
git clone git://git.buildroot.net/buildroot -b 2024.02.2
```

## Buildroot Build vorbereiten

Ähnlich wie zuvor kann mit 'list-defconfigs' die Default Configs aufgelistet  
werden. In meinem Fall habe ich die raspi64 Config ausgewählt.
```
cd buildroot
make list-defconfigs
make raspberrypi4_64_defconfig
```

Damit man sich später auch mit ssh aufschlaten kann, muss hierzu noch ein SSH  
Server mit installiert werden.

```
$ make menuconfig
Target packages submenu --> Networking applications
dropbear
```

Zum Schluss das Image noch kompilieren. Das kann erneut einige Minuten dauern.  
Hierzu werden die ganzen Packete auf dem Host heruntergeladen (ca. 12GB!).
```
make
```

Das fertige Image liegt dann unter folgendem Pfad
```
output/images/sdcard.img
```

Und nun das Image auf die SD Karte flashen, hierzu den zuvor installierten  
balenaEtcher verwenden:
```
1. Insert a microSD card into your host machine
2. Launch Etcher
3. Click Flash from file from Etcher
4. Locate the .wic image that you built for the Raspberry Pi 4 and open it
5. Click Select target from Etcher
6. Select the microSD card that you inserted in Step 1
7. Click Flash from Etcher to write the image
8. Eject the microSD card when Etcher is done flashing
9. Insert the microSD card into your Raspberry Pi 4
10. Apply power to the Raspberry Pi 4 by way of its USB-C port
```

Nun kann entweder über die serielle Schnittstelle oder über SSH auf den Pi  
zugegriffen werden. Hierbei sei gesagt, dass ich mich beim ersten Mal nicht  
über SSH verbinden konnte, da kein root Passwort gesetzt war. Nach einmaligen  
Verbinden via Konsole und setzen eines root Passworts, hat die Verbindung über  
SSH ebenfalls funktioniert.


# Build System - Yocto Project
## Yocto Repo clonen
Zuerst auf der Release Seite von Yocto nach der aktuellen LTS Version schauen  
https://wiki.yoctoproject.org/wiki/Releases

Stand jetzt ist dies neuste Version:  
| Codename              | Scarthgap                            |
|-----------------------|--------------------------------------|
| Yocto Project Version | 5.0                                  |
| Release Date          | April 2024                           |
| Support Level         | Long Term Support (until April 2028) |

Allerdings hat der Layer 'meta-raspberrypi' Stand heute **noch keinen Release**  
dafür. Deshalb werde ich im Folgendem die vorherige LTS Version verwenden:  
Kirkstone.


Danach mit diesem Befehl und dem angepasstem Branch Namen das Repo  
herunterladen:
```
git clone -b kirkstone git://git.yoctoproject.org/poky.git
```

## Yocto Build vorbereiten

Build Verzeichnis erstellen
```
source poky/oe-init-build-env build-rpi
```

**Optional:** Sollte in der Zwischenzeit der Host Rechner neugestartet werden,  
muss das Build Verzeichnis erneut "ausgewählt" werden:
```
cd yocto
source poky/oe-init-build-env build-rpi
```

Layers herunterladen
```
git clone -b kirkstone git://git.openembedded.org/meta-openembedded
git clone -b kirkstone git://git.yoctoproject.org/meta-raspberrypi
```

Layers zur Config hinzufügen
```
cd build-rpi
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-multimedia
bitbake-layers add-layer ../meta-openembedded/meta-networking
bitbake-layers add-layer ../meta-raspberrypi
```

Die Datei 'bblayers.conf' sollte nun folgende Layers beinhalten:
```
$ cd build-rpi
$ bitbake-layers show-layers

layer                 path                                      priority
==========================================================================
meta                  /home/stephan/Documents/yocto/poky/meta   5
meta-poky             /home/stephan/Documents/yocto/poky/meta-poky  5
meta-yocto-bsp        /home/stephan/Documents/yocto/poky/meta-yocto-bsp  5
meta-oe               /home/stephan/Documents/yocto/meta-openembedded/meta-oe  5
meta-python           /home/stephan/Documents/yocto/meta-openembedded/meta-python  5
meta-multimedia       /home/stephan/Documents/yocto/meta-openembedded/meta-multimedia  5
meta-networking       /home/stephan/Documents/yocto/meta-openembedded/meta-networking  5
meta-raspberrypi      /home/stephan/Documents/yocto/meta-raspberrypi  9
```

Nun muss noch die 'local.conf' Datei angepasst werden, hier können verschiedene  
Einstellungen getroffenen werden.  
Im Folgendem die relevanten:
```
MACHINE ??= "raspberrypi4-64"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks ssh-server-openssh"
ENABLE_UART = "1"
```


Mit folgendem Befehl kann der Build Prozess getestet werden, um zu schauen ob  
alles korrekt konfiguriert ist.  
**In diesem Schritt wird noch nichts ausgeführt!**
```
$ cd build-rpi
$ bitbake rpi-test-image -n

Build Configuration:
BB_VERSION           = "2.0.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "linuxmint-21.3"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "raspberrypi4-64"
DISTRO               = "poky"
DISTRO_VERSION       = "4.0.18"
TUNE_FEATURES        = "aarch64 armv8a crc cortexa72"
TARGET_FPU           = ""
meta                 
meta-poky            
meta-yocto-bsp       = "kirkstone:31751bba1c789f15f574773a659b8017d7bcf440"
meta-oe              
meta-python          
meta-multimedia      
meta-networking      = "kirkstone:5a6f7925bd2b885955c942573f70a5594f231563"
meta-raspberrypi     = "kirkstone:9dc6673d41044f1174551120ce63501421dbcd85"
```


Sofern alles korrekt konfiguriert ist, können nun alle benötigten Packete  
heruntergeldaen werden: (ca. 90GB!)
```
$ cd build-rpi
$ bitbake rpi-test-image --runonly=fetch

ARNING: Host distribution "linuxmint-21.3" has not been validated with this version of the build system; you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
Loading cache: 100% |########################################################################################| Time: 0:00:00
Loaded 3934 entries from dependency cache.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.0.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "raspberrypi4-64"
DISTRO               = "poky"
DISTRO_VERSION       = "4.0.18"
TUNE_FEATURES        = "aarch64 armv8a crc cortexa72"
TARGET_FPU           = ""
meta                 
meta-poky            
meta-yocto-bsp       = "kirkstone:31751bba1c789f15f574773a659b8017d7bcf440"
meta-oe              
meta-python          
meta-multimedia      
meta-networking      = "kirkstone:5a6f7925bd2b885955c942573f70a5594f231563"
meta-raspberrypi     = "kirkstone:9dc6673d41044f1174551120ce63501421dbcd85"

Initialising tasks: 100% |###################################################################################| Time: 0:00:01
Sstate summary: Wanted 0 Local 0 Mirrors 0 Missed 0 Current 0 (0% match, 0% complete)
NOTE: No setscene tasks
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 322 tasks of which 0 didn't need to be rerun and all succeeded.

Summary: There was 1 WARNING message.
```



Im Anschluss kann wie folgt der eigentliche Build Prozess gestartet werden:  
(dies kann mehrere Minuten :) dauern!!)
```
cd build-rpi
bitbake rpi-test-image
```

Das fertige Image liegt nun im Verzeichnis 'tmp/deploy/images/raspberrypi4-64'
```
$ find . -type f -name "rpi-test-image*"

./rpi-test-image-raspberrypi4-64-20240430113712.testdata.json
./rpi-test-image-raspberrypi4-64-20240430113712.rootfs.ext3
./rpi-test-image-raspberrypi4-64-20240430113712.rootfs.manifest
./rpi-test-image-raspberrypi4-64-20240430113712.rootfs.wic.bmap
./rpi-test-image-raspberrypi4-64-20240430113712.rootfs.wic.bz2
./rpi-test-image.env
./rpi-test-image-raspberrypi4-64-20240430113712.rootfs.tar.bz2
```

Nun noch die WIC Datei extrahieren
```
$ cd tmp/deploy/images/raspberrypi4-64
$ bzip2 -d -f rpi-test-image-raspberrypi4-64.wic.bz2

$ find . -type f -name "rpi-test-image-raspberrypi4-64.wic"

./rpi-test-image-raspberrypi4-64.wic
```

Und nun das Image auf die SD Karte flashen, hierzu den zuvor installierten  
balenaEtcher verwenden:
```
1. Insert a microSD card into your host machine
2. Launch Etcher
3. Click Flash from file from Etcher
4. Locate the .wic image that you built for the Raspberry Pi 4 and open it
5. Click Select target from Etcher
6. Select the microSD card that you inserted in Step 1
7. Click Flash from Etcher to write the image
8. Eject the microSD card when Etcher is done flashing
9. Insert the microSD card into your Raspberry Pi 4
10. Apply power to the Raspberry Pi 4 by way of its USB-C port
```

Default Login Credentials:
```
user: root
password: *empty*
```