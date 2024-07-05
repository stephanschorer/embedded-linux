# Verwendete Anleitungen
https://hechao.li/2021/12/20/Boot-Raspberry-Pi-4-Using-uboot-and-Initramfs/  
https://danmc.net/posts/raspberry-pi-4-b-u-boot/  
https://www.96boards.org/blog/boot-linux-from-sd-card-uboot/  
https://community.arm.com/oss-platforms/w/docs/495/tftp-remote-network-kernel-using-u-boot  


# U-Boot

## Requirements installieren

```
sudo apt-get install git make patch u-boot-tools
```

## SD Karte vorbereiten

```
sudo fdisk /dev/sdb
```
Nun müssen dort zwei Partitionen, mit "n", erstellt werden


| Partition number | type | size |
|------------------|------|------|
| 1                | boot | 1GiB |
| 2                | root | rest |

Anschließend muss der Partitionstyp von der ersten Partition "1" auf mit dem  
Kommand "t" auf "W95 FAT32" geändert werden.  
Die Partitionstabelle kann danach mit "w" abgespeichert werden.

Anschließend müssen die Partitionen formatiert werden
```
sudo mkfs.vfat -F 32 -n boot /dev/sdb1
sudo mkfs.ext4 -L root /dev/sdb2
```

## U-Boot erstellen

Zunächst das Repo klonen:
```
git clone git://git.denx.de/u-boot.git
$ cd u-boot
```
Cross Compiler in Variable schreiben:
```
export CROSS_COMPILE=aarch64-rpi4-linux-gnu-
```

Im Unterordner sind erneut wie bei crosstool-ng samples vorhanden:
```
./u-boot/configs/rpi_4_defconfig
```

Zunächst muss die Basiskonfiguration gesetzt werden
```
make rpi_4_defconfig
```

Damit ich u-boot kompilieren konnte, musste ich zunächst folgedes Paket  
installieren.
```
apt-get install libssl-dev
```

Ansonsten hatte ich diese Fehlermeldung:
```
In file included from tools/imagetool.h:24,
                 from tools/aisimage.c:7:
include/image.h:1471:12: fatal error: openssl/evp.h: No such file or directory
 1471 | #  include <openssl/evp.h>
      |            ^~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.host:112: tools/aisimage.o] Error 1
make: *** [Makefile:1892: tools] Error 2
```
U-Boot fertig kompilieren
```
make
```

## Dateien auf SD Karte kopieren

u-boot kopieren
```
sudo cp u-boot.bin /media/stephan/boot/
```

Da der Raspi4 einen proprietären Bootloader hat, muss dieser u-boot booten.  
Hierzu müssen folgende Dateien vom Raspi Github in die boot Partition kopiert  
werden.  
https://github.com/raspberrypi/firmware/blob/master/boot
```
start4.elf
fixup4.dat
bootcode.bin
```

Anschließend eine config.txt Datei mit folgendem Inhalt erstellen und nach  
boot/ kopieren:
```
cat << EOF > config.txt
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
EOF
sudo mv config.txt /media/stephan/boot/
```

Mit "kernel=u-boot.bin" wird dem integriertem Bootloader mitgegeben, dass er  
doch u-boot booten soll.  

Die Boot Partition sollte nun folgende Dateien beinhalten:
```
boot/
|-- start4.elf
|-- fixup4.dat
|-- bootcode.bin
|-- config.txt
|-- u-boot.bin
```

Anschließend kann die SD-Karte im Pi eingesteckt und gestartet werden.


## u-boot navigieren

Gesetzte Variablen anzeigen:
```
printenv
```

Neue Variablen erstellen:
```
setenv newvar value
```
Gesetzte Variablen löschen
```
setenv newvar 
```