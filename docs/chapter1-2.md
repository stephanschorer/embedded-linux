# Raspberry Pi über Serielle Schnittstelle erreichen

## Raspberry Pi konfigurieren
Zuerst habe ich es mit wsl2 unter win11 versucht, dies hat aber zu mehreren  
Problemen geführt, da ich die Serielle Schnittstelle nicht korrekt zu wsl2  
weiterleiten konnte.
https://www.youtube.com/watch?v=fz0B3kPjGlQ

Deshalb hab ich mich doch entschieden auf meinem Laptop eine Linux Distro zu  
installieren (Dual Boot).

Zuerst muss auf dem Pi "UART" aktiviert werden:
```
sudo raspi-config
```
Dort zum Untermenü "Interfaces" und "Serielle Schnittstelle" navigieren  
und aktivieren.

Anschließend sollte in folgender Datei "UART" aktiviert sein.
(/boot/firmware/config.txt)
```
enable_uart=1
```

Danach den Pi rebooten.

## TTL to USB Kabel an GPIO anstecken
Pin Layout ist in der Doku zu finden:  
https://www.raspberrypi.com/documentation/computers/raspberry-pi.html

TTL to USB Layout:  
https://www.amazon.de/dp/B07Z7PPT6Y?psc=1&ref=ppx_yo2ov_dt_b_product_details

Pi GPIO Pin9 (Ground) --> USB Black (Ground)  
Pi GPIO Pin8 (TXD) --> USB White (Rx)  
Pi GPIO Pin10 (RXD) --> USB White (Tx)  


## Minicom konfigurieren
Um auf die serielle Schnittstelle zuzugreifen verwende ich minicom.

Zuerst muss der Client noch konfiguriert werden.
```
minicom --setup
```

Unter "Serial port setup" müssen folgende Einstellungen geändert werden:
| Options               | new Value    |
|-----------------------|--------------|
| Serial Device         | /dev/ttyUSB0 |
| Hardware Flow Control | No           |

Nach erfolgreicher Konfiguration kann über diesen Befehl direkt auf den Pi  
zugegriffen werden. Es sind keine weiteren Parameter notwendig.
```
minicom
```

# GCC Cross Toolchain einrichten

## crosstool-ng installieren

Installieren der benötigten Packete
```
sudo apt-get install autoconf automake bison bzip2 cmake flex g++ gawk gcc
gettext git gperf help2man libncurses5-dev libstdc++6 libtool libtool-bin make
patch python3-dev rsync texinfo unzip wget xz-utils
```

Aktuelle crosstool-ng Version herunterladen und entpacken
```
wget http://crosstool-ng.org/download/crosstool-ng/crosstool-ng-1.26.0.tar.xz
tar xf crosstool-ng-1.26.0.tar.xz
cd crosstool-ng-1.26.0
```

configure, make, make install procedure
```
./configure --prefix=/opt/crosstoolng
make
make install
```

PATH Variable anpassen (~/.bashrc)
```
export PATH="${PATH}:/opt/crosstoolng/bin"
```

## PI Details sammeln

Da der Linker, einer erstellte Toolchains mit crosstool-ng, jedoch nicht über  
die erforderlichen Bibliothekssuchpfade verfügt, muss bei der Erstellung der  
Patch für binutils mitgegeben werden.  
Hierzu muss auf dem PI folgendes installiert werden
```
apt-get install binutils-source
```

Anschließend ist unter /usr/src/binutils/patches/ der benötigte Patch zu  
finden, welcher später auf dem Host benötigt wird.
```
129_multiarch_libpath.patch
scp root@raspberrypi:/usr/src/binutils/patches/129_multiarch_libpath.patch /root/
```

Nun werden noch einige Versions Nummern vom PI benötigt
```
uname -r
6.6.20+rpt-rpi-v8

ld --version
GNU ld (GNU Binutils for Debian) 2.40

gcc --version
gcc (Debian 12.2.0-14) 12.2.0

ldd --version
ldd (Debian GLIBC 2.36-9+rpt2+deb12u4) 2.36
```

Dieser Schritt ist schätzungsweise, wenn überhaupt erst später notwendig,  
trotzdem habe ich ihn bereits jetzt durchgefürht.  
Dies ist notwendig, da später einige Dateien vom PI zum Host kopiert werden und  
ansonsten die Symlinks nicht mehr funktionieren würden.
```
apt-get install symlinks
symlinlks -rc /.
```

## Arbeitsbereich vorbereiten

Zuerst ein Arbeitsverzeichnis erstellen, in welchem die Toolchain später  
konfiguriert und abgelegt wird
```
mkdir workdir
cd workdir
mkdir src
mkdir -p patches/binutils/2.40
cp /root/129_multiarch_libpath.patch workdir/patches/binutils/2.40/
```

## toolchain konfigurieren und erstellen

Mit folgendem Befehl kann man sich Beispiele anzeigen lassen.
```
ct-ng list-samples
aarch64-rpi4-linux-gnu
```

Anschließend hiermit die Vorkonfiguration auswählen und das Menü für weitere  
notwendige Konfigurationen öffnen
```
ct-ng aarch64-rpi4-linux-gnu
ct-ng menuconfig
```

Im Menü habe ich einige Einstellungen vorgenommen, welche im Folgendem kurz  
aufgelistet sind:

### paths and misc options
| Options                             | new Value                                              |
|-------------------------------------|--------------------------------------------------------|
| local tarballs directory            | ${CT_TOP_DIR}/src                                      |
| working directory                   | ${CT_TOP_DIR}/build                                    |
| prefix directory                    | ${CT_TOP_DIR}/\${CT_HOST:+HOST-\${CT_HOST}/}\${CT_TARGET} |
| remove prefix dir prior to building | enable                                                 |
| Render the toolchain read-only      | disable                                                |
| Strip target toolchain executables  | enable                                                 |
| Patches origin                      | Bundled, then local                                    |
| Local patch directory               | ${CT_TOP_DIR}/patches                                  |

### target options
| Options               | new Value  |
|-----------------------|------------|
| Emit assembly for CPU | cortex-a72 |

### toolchain options
| Options                | new Value |
|------------------------|-----------|
| Build Static Toolchain | enable    |

### operating system options
| Options          | new Value |
|------------------|-----------|
| Version of Linux | 6.4       |

### binary utilties options
| Options                | new Value |
|------------------------|-----------|
| Version of binaryutils | 2.40      |

### C-library options
| Options          | new Value |
|------------------|-----------|
| Version of glibc | 2.36      |

### C-compiler options
| Options          | new Value          |
|------------------|--------------------|
| Version of gcc   | 12.3.0             |
| gcc extra config | --enable-multiarch |
| C++              | enable             |

Anschließend mit 'Save' die vorgenommen Einstellungen in der Config File  
'.config' speichern

Als letzten Schritt muss noch eine Variable gesetzt werden
```
export DEB_TARGET_MULTIARCH=arm-linux-gnueabihf
```

Nun kann die Toolchain erstellt werden dies hat bei mir meistens 15min gedauert
```
ct-ng build
```

Sobald dies abgeschlossen ist kann mit folgendem Befehl noch getestet werden,  
ob die Patches richtig angewandt wurden
```
aarch64-rpi4-linux-gnu/aarch64-rpi4-linux-gnu/bin/ld --verbose \
| grep -i "search"
```
Der Output sollte folgendes beinhalten:  
SEARCH_DIR("=/lib/arm-linux-gnueabihf");
SEARCH_DIR("=/usr/lib/arm-linux-gnueabihf");

PATH Variable anpassen (~/.bashrc)
```
export PATH="${PATH}:/home/stephan/Documents/crosstoolng/workdir/aarch64-rpi4-linux-gnu/bin"
```

## crosstool-ng testen

Folgendes helloworld.c
```
#include <stdio.h>
#include <stdlib.h>
int main (int argc, char *argv[])
{
    printf ("Hello, world!\n");
    return 0;
}
```

kann nach erfolgreicher Erstellung der Toolchain mit diesem Befehl für die  
PI vorbereitet und kompiliert werden.
```
aarch64-rpi4-linux-gnu-gcc helloworld.c -o helloworld
```

Bei Bedarf kann mit file noch die Datei genauer analysiert werden
```
file helloworld
helloworld: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV),
dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 6.4.0
with debug_info, not stripped
```