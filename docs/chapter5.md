# Verwendete Anleitungen
https://github.com/mhomran/u-boot-rpi3-b-plus?tab=readme-ov-file  
https://hechao.li/2021/12/20/Boot-Raspberry-Pi-4-Using-uboot-and-Initramfs/  

# Root Filesystem
## rootfs Struktur erstellen

```
mkdir rootfs
cd rootfs
mkdir {bin,dev,etc,home,lib64,proc,sbin,sys,tmp,usr,var}
mkdir usr/{bin,lib,sbin}
mkdir var/log
```

Symbolischer Link 'lib' erstellen, welcher auf 'lib64' zeigt
```
ln -s lib64 lib
```

Berechtigungen auf 'root' user ändern
```
sudo chown -R root:root *
```

## Busybox runterladen und konfigurieren

Repo clonen und aktuelle version auswählen
```
git clone git://busybox.net/busybox.git
cd busybox
git checkout 1_36_stable
```

Crosscompiler in Variable schreiben und Default Config auswählen
```
CROSS_COMPILE=aarch64-rpi4-linux-gnu-
make CROSS_COMPILE="$CROSS_COMPILE" defconfig
```

Installations Verzeichnis auf rootfs Order anpassen
```
sed -i 's%^CONFIG_PREFIX=.*$%CONFIG_PREFIX="/home/stephan/Documents/rootfs/"%' .config
```

Busybox erstellen
```
make CROSS_COMPILE="$CROSS_COMPILE"
```

## Busybox auf rootfs installieren

Busybox auf rootfs installieren, 'sudo' weil root der Owner vom 'rootfs'  
Verzeichnis ist
```
CROSS_COMPILE=/home/stephan/Documents/crosstool-ng/workdir/aarch64-rpi4-linux-gnu/bin/aarch64-rpi4-linux-gnu-
sudo make CROSS_COMPILE="$CROSS_COMPILE" install
```

Benötigte Bibliotheken für Busybox herausfinden
```
cd rootfs
readelf -a bin/busybox | grep -E "(program interpreter)|(Shared library)"

      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libresolv.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```

Besagte Bibliotheken nach rootfs kopieren
```
cd rootfs
export SYSROOT=$(aarch64-rpi4-linux-gnu-gcc -print-sysroot)
sudo cp -L ${SYSROOT}/lib64/{ld-linux-aarch64.so.1,libm.so.6,libresolv.so.2,libc.so.6} lib64/
```

Folgende Dateien sollten nun in 'lib64' liegen
```
lib64/
├── ld-linux-aarch64.so.1
├── libc.so.6
├── libm.so.6
└── libresolv.so.2
```

Diese zwei Device Nodes werden von Busybox benötigt
```
cd rootfs
sudo mknod -m 666 dev/null c 1 3
sudo mknod -m 600 dev/console c 5 1
```

## rootfs Struktur auf SD Karte übertragen

Beide Partitionen der SD Karte mounten (sofern kein automount)

lokal erstelltes rootfs in die root Partition kopieren
```
sudo cp -r rootfs/* /media/stephan/root/
```

Die vorhandene cmdline.txt (für u-boot) muss nicht angepasst werden, da die  
dort gesetzten Argumente durch die 'bootargs' überschrieben werden siehe [Quelle](https://danmc.net/posts/raspberry-pi-4-b-u-boot/#:~:text=%23%20The%20combined%20args%20are%20used%20to%20boot%20the%20kernel.%20If%20%22setenv%20bootargs%20...%22%0A%23%20is%20used%20in%20the%20u%2Dboot%20boot%20script%2C%20it%20will%20override%20the%20device%20tree%20bootargs%0A%23%20and%20the%20bootargs%20in%20cmdline.txt.):
```
# The combined args are used to boot the kernel. If "setenv bootargs ..."
# is used in the u-boot boot script, it will override the device tree bootargs
# and the bootargs in cmdline.txt.
```

bootargs für u-boot erstellen und auf SD Karte kopieren
```
cd u-boot
cat << EOF > boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} uImage
setenv bootargs "8250.nr_uarts=1 console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootwait"
bootm \${kernel_addr_r} - \${fdt_addr}
EOF

tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr
sudo cp boot.scr /media/stephan/boot/
```

Nun sollte der Pi zuerst u-boot laden, welches anschließend die boot.scr File  
automatisch erkennt und somit den Kernel bootet und das rootfs mountet.

**Sämtliche Änderungen im Folgenden müssen natürlich jedesmal auf die SD Karte**  
**übertragen werden!**

## Das init Programm

Folgende Datei ist die Konfiguration File für das integrierte Busybox init  
program.  
```
cat << EOF > /etc/inittab
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/ash
::respawn:/sbin/syslogd -n
::respawn:/sbin/klogd -n
EOF
```

In der File 'rcS' können dann jegliche Befehle geschrieben werden, welche beim  
Start ausgeführt werden sollen.  
Wichtig da es sich um ein Script handelt, **muss die Datei ausführbar sein**!
```
cat << EOF > /etc/init.d/rcS
#!/bin/sh
mount -t proc none /proc
mount -o remount,rw /
mount -t sysfs none /sys
EOF
chmod +x /etc/init.d/rcS
```

Damit das init program bei Start auch ausgeführt wird, müssen die bootargs  
nochmal angepasst werden
```
setenv bootargs "bootargs + rdinit=/sbin/init"
```
Bei Bedarf kann dies auch erneut in das boot.scr integriert werden, das habe  
ich jedoch bereits weiter oben beschrieben.

## Erstellung von User Accounts

Zum einen wird folgende Datei benötigt, in welcher sämtliche Benutzer definiert  
sind.
```
cat << EOF > /etc/passwd
root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/false
EOF
```
Da hierauf jedoch sämtliche Dienste Zugriff benötigen (aka chmod 644), ist es  
nicht sinnvoll das Passwort direkt in dieser Datei abzuspeichern.  
Genau deshalb gibt es eine weitere Datei, in welcher die Passwörter  
abgespeichert werden können, hierzu muss in der '/etc/passwd' an zweiter Stelle  
das 'x' gesetzt werden.

Im zweiten File wird das besagte gehashte Passwort abespeichert.  
Beim root Benutzer ist aktuell keins hinterlegt, sprich es wird kein Passwort  
für den Login benötigt.
```
cat << EOF > /etc/shadow
root::10933:0:99999:7:::
daemon:*:10933:0:99999:7:::
EOF
chmod 600 /etc/shadow
```

Damit sich Benutzer ab Systemstart anmelden können, muss dafür gesorgt werden,  
dass 'getty' gespawnt wird.  
Hierzu muss folgendes Zeile in der /etc/inittab hinzugefügt werden
```
::respawn:/sbin/getty 115200 console
```
Zudem sollte folgende Zeile entfernt werden:
```
::askfirst:-/bin/ash
```
**Warum muss '/bin/ash' entfernt werden?**  
'/bin/ash': sorgt dafür, dass eine Shell ohne Login bei Systemstart spawnt  
'/sbin/getty': hingegen promptet den Benutzer nach einem Benutzernamen  
Bitte nicht beide in '/etc/inittab' rein schreiben :)

Natürlich können im gebootetem Zustand, wie gewohnt, die Passwörter auch  
angepasst werden: (in meinem Fall)
```
adduser test
passwd test embeddedlinux
```
Anschließend wird auch die Datei '/etc/shadow' aktualisiert und dort das  
gehashte Passwort abgespeichert.
```
test:6FmhWLFsWJ5Fk:1:0:99999:7:::
```
Zudem wird die Zeile in der Datei '/etc/passwd' hinzugefügt:
```
test:x:1000:1000:Linux User,,,:/home/test:/bin/sh
```

## Rootfs über Netzwerk mounten
Zuerst muss auf dem Host der NFS Server installiert werden, worüber später das rootfs geteilt wird.
```
sudo apt install nfs-kernel-server
```

Nun muss der entsprechende Pfad konfiguriert werden:
```
$ cat /etc/exports

/home/stephan/Documents/u-boot/rootfs *(rw,sync,no_subtree_check,no_root_squash)
```

Danach den NFS Server neustarten, damit die Änderungen übernommen werden.
```
systemctl restart nfs
```
Nun ist das rootfs über das Netzwerk erreichbar. Im nächsten Schritt muss dem Pi der NFS Server mitgegeben werden. Dies erfolgt ähnlich wie zuvor bei der lokalen Variante.

Hierzu muss die Kernel Config (boot_cmd.txt) angepasst werden:
```
cd u-boot
$ cat boot_cmd.txt
fatload mmc 0:1 \${kernel_addr_r} uImage
setenv bootargs "8250.nr_uarts=1 console=ttyS0,115200n8 root=/dev/nfs rw nfsroot=192.168.178.74:/home/stephan/Documents/u-boot/rootfs ip=192.168.178.100 rootwait"
bootm \${kernel_addr_r} - \${fdt_addr}
EOF

tools/mkimage -A arm64 -O linux -T script -C none -d boot_cmd.txt boot.scr
sudo cp boot.scr /media/stephan/boot/
```
Danach wird der Kernel vom lokalem Kernel Image 'uImage' gebootet und danach wird das rootfs vom Host Computer gemountet.
Gleich wie beim lokalen rootfs sind jegliche Änderungen persistent.