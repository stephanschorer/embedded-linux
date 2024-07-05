# Verwendete Anleitungen
https://hechao.li/2021/12/20/Boot-Raspberry-Pi-4-Using-uboot-and-Initramfs/  

# Raspi Kernel
## raspi4 Kernel kompilieren

Zunächst das Repo clonen
```
git clone https://github.com/raspberrypi/linux.git
cd linux
```

Anschließend den Kernel kompilieren
```
make ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu- bcm2711_defconfig
make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-rpi4-linux-gnu-
```

Anschließend sind im Verzeichnis "arch/arm64/boot/" die benötigen Dateien  
erstellt worden.
```
total 50124
drwxrwxr-x  3 stephan stephan     4096 Apr 13 18:11 .
drwxrwxr-x 12 stephan stephan     4096 Apr 13 16:34 ..
drwxrwxr-x 33 stephan stephan     4096 Apr 13 16:03 dts
-rw-rw-r--  1 stephan stephan 21721600 Apr 13 18:10 Image
```

## Dateien auf SD-Karte kopieren

Die "Image" Datei muss nun noch für u-boot vorbereitet und wie gehabt auf boot/  
kopiert werden:
```
mkimage -A arm64 -O linux -C none -T kernel -a 0x80080000 -e 0x80080000 \
-n raspi4kernel -d arch/arm64/boot/Image uImage
sudo cp uImage /media/stephan/boot/
```

Alternativ kann der Pi auch ohne u-boot den Kernel laden hierzu muss zusätzlich  
das Image als kernel8.img in boot/ abgelegt werden
```
cp arch/arm64/boot/Image /media/stephan/boot/kernel8.img
```

Im Verzeichnis "dts/broadcom" ist zudem noch der Device Tree zufinden.  
Hier ebenfalls den benötigten nach boot/ kopieren.
```
sudo cp arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /media/stephan/boot/
```

Anschließend eine cmdline.txt Datei mit folgendem Inhalt erstellen und nach  
boot/ kopieren:
```
cat << EOF > cmdline.txt
console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootwait
EOF
sudo mv cmdline.txt /media/stephan/boot/
```

Die Boot Partition sollte nun folgende Dateien beinhalten:
```
boot/
|-- start4.elf
|-- fixup4.dat
|-- bootcode.bin
|-- config.txt
|-- cmdline.txt
|-- uImage
|-- kernel8.img
|-- u-boot.bin
```

## raspi4 Kernel mit integriertem Bootloader starten 
Sofern statt u-boot der integrierte Bootloader verwendet werden soll, muss in  
der config.txt die letzte Zeile entfernt werden.  
```
enable_uart=1
arm_64bit=1
>kernel=u-boot.bin<
```
Sofern der Raspi beim Bootvorgang den Kernel (kernel8.img) im boot/ findet wird  
automatische dieser geladen.  

## raspi4 Kernel mit u-boot starten 

### Booten von SD-Karte

In der u-boot Shell angelangt kann mit folgendem Befehl das Kernel Image in  
Memory geladen werden
```
# Prüfen ob richtiges Verzeichnis ausgewählt ist
ls mmc 0:1
fatload mmc 0:1 ${kernel_addr_r} uImage
```

### Booten über Netzwerk

TFTP Daemon auf Host installieren
```
sudo apt-get update && sudo apt-get install xinetd tftpd tftp
```

Anschließend Konfiguration Datei und Share Verzeichnis erstellen
```
cat << EOF > /etc/xinetd.d/tftp
service tftp
{
     protocol = udp
     port = 69
     socket_type = dgram
     wait = yes
     user = nobody
     server = /usr/sbin/in.tftpd
     server_args = /mnt/tftp
     disable = no
}
EOF

sudo mkdir /mnt/tftp
sudo chmod -R 777 /mnt/tftp
sudo chown -R nobody /mnt/tftp
sudo service xinetd restart
```

Nun noch das uImage in den TFTP Share kopieren
```
cp arch/arm64/boot/uImage /mnt/tftp/
```

Anschließend kann die u-boot Shell auf den TFTP Service wie folgt zugreifen:
```
setenv ipaddr 192.168.178.100
setenv serverip 192.168.178.74

tftp ${kernel_addr_r} uImage   

Using ethernet@7d580000 device
TFTP from server 192.168.178.74; our IP address is 192.168.178.100
Filename 'uImage'.
Load address: 0x1000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         ##################  0 Bytes
         599.6 KiB/s
done
Bytes transferred = 21721664 (14b7240 hex)
```

Hiermit kann im Anschluss das Image im Memory angeschaut werden
```
iminfo ${kernel_addr_r}

## Checking Image at 00080000 ...
   Legacy image found
   Image Name:   raspi4kernel
   Image Type:   AArch64 Linux Kernel Image (uncompressed)
   Data Size:    21721600 Bytes = 20.7 MiB
   Load Address: 80080000
   Entry Point:  80080000
   Verifying Checksum ... OK
```


Mit diesem Befehl kann nun der Kernel im Memory geladen werden.  
Dieser Prozess stoppt am Ende mit "end Kernel panic", da das rootfs nicht  
gefunden wurde
```
bootm ${kernel_addr_r} - ${fdt_addr}

[    3.083620] Call trace:
[    3.086113]  dump_backtrace+0x0/0x1e0
[    3.089835]  show_stack+0x24/0x30
[    3.093205]  dump_stack+0xe8/0x150
[    3.096658]  panic+0x188/0x380
[    3.099761]  kernel_init+0x110/0x130
[    3.103391]  ret_from_fork+0x10/0x38
[    3.107026] SMP: stopping secondary CPUs
[    3.111015] Kernel Offset: disabled
[    3.114556] CPU features: 0x8240022,61002000
[    3.118888] Memory Limit: none
[    3.122000] ---[ end Kernel panic - not syncing: No working init found.
```