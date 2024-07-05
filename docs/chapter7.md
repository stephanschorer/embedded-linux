# Verwendete Anleitungen

# mit Pi verbinden
Da wir in der vorherigen Anleitung SSH und UART aktiviert haben, können wir uns  
nun über beide Wege auf den Pi schalten
```
EXTRA_IMAGE_FEATURES ?= "ssh-server-openssh"
ENABLE_UART = "1"
```

```
#SSH
ssh root@raspberrypi4-64

#UART (wie gehabt)
minicom
```

# Wifi testen

Zuerst die connmanctl starten, worüber das Interface konfiguriert werden kann.
```
connmanctl
```

Mit folgenden Befehlen kann Wifi aktiviert und die Umgebung gescannt werden.  
Der Dritte listet die gefundenen Netwerke auf.
```
$ connmanctl> enable wifi
$ connmanctl> agent on
$ connmanctl> scan wifi
$ connmanctl> services
*AR Wired                ethernet_dca63251818c_cable
    VR-Network           wifi_dca63251818d_*_managed_psk
    Vodafone-Router      wifi_dca63251818d_*_managed_psk
```

Anschließend kann man sich hiermit mit einem Netzwerk verbinden, wobei der '*'  
die Zahlenbuchstabenkomination von oben darstellt
```
$ connmanctl> connect wifi_dca63251818d_*_managed_psk
Agent RequestInput wifi_dca63251818d_*_managed_psk
  Passphrase = [ Type=psk, Requirement=mandatory ]
Passphrase? *enterSuperSecretPassword*
Connected wifi_dca63251818d_*_managed_psk
```

Wenn nun erneut der 'services' Befehl ausgeführt wird, sieht man nun, dass der  
Pi mit dem zuvor definiertem Netzwerk verbunden ist '*AR'
```
connmanctl> services
*AR Wired                ethernet_dca63251818c_cable
*AR Vodafone-Router      wifi_dca63251818d_*_managed_psk
    VR-Network           wifi_dca63251818d_*_managed_psk
```

Alternativ ein Ethernet Kabel anstecken.

# Custom Layer hinzufügen

Environment Variablen für build Pfad setzten
```
source poky/oe-init-build-env build-rpi
```

Neuen Layer erstellen und Ordner/Dateien wie folgt umbenennen:
```
cd build-rpi
bitbake-layers create-layer ../meta-gattd
mv recipes-example recipes-gattd
cd recipes-gattd
mv example gattd
cd gattd
mv example_0.1.bb gattd_0.1.bb
```

Das Verzeichnis sollte nun so aussehen:
```
.
├── conf
│   └── layer.conf
├── COPYING.MIT
├── README
└── recipes-gattd
    └── gattd
        └── gattd_0.1.bb
```

Nun kann einfacherhalber aus [meta-gattd](https://github.com/PacktPublishing/Mastering-Embedded-Linux-Programming-Third-Edition/tree/master/Chapter07/meta-gattd) der Code in das lokale Verzeichnis kopiert werden.  
**Wichtig** hier müssen die verwendeten LTS Versionen angepasst werden!  
(zufinden in der layer.conf Datei)

Nun den erstellen Layer zum Image hinzufügen
```
cd build-rpi
bitbake-layers add-layer ../meta-gattd
```

Nun erneut die conf/local.conf Datei wie folgt anpassen:
```
CORE_IMAGE_EXTRA_INSTALL += "gattd"
```

Im Anschluss das Image neuerstellen. Für genauere Schritte siehe [Chapter 6](chapter6.md)
```
bitbake rpi-test-image
```

# Eigene Distro

## Vorkehrungen
Umgebungsvariablen setzten
```
source poky/oe-init-build-env build-rpi
```

Zu Beginn die zuvor angepassten Settings für gattd wieder entfernen.
```
bitbake-layers remove-layer ../meta-gattd
```

Folgende Zeile in conf/local.conf wieder auskommentieren
```
#CORE_IMAGE_EXTRA_INSTALL += "gattd"
```

## Distro Layer erstellen

Zuerst einen neuen Layer erstellen und diesem direkt im Image verankern.
```
bitbake-layers create-layer ../meta-mackerel
bitbake-layers add-layer ../meta-mackerel
```

Config Datei für Distro erstellen
```
cd meta-mackerel
mkdir -p conf/distro
touch conf/distro/mackerel.conf
```

mackerel.conf Datei wie folgt anpassen
```
DISTRO_NAME = "Mackerel (Mackerel Embedded Linux Distro)"
DISTRO_VERSION = "0.1"
#DISTRO_FEATURES: Add software support for these features.
#DISTRO_EXTRA_RDEPENDS: Add these packages to all images.
#DISTRO_EXTRA_RRECOMMENDS: Add these packages if they exist.
#TCLIBC: Select this version of the C standard library.
```

Dieser Layer kann nun für Einstellungen verwendet werden.  
Für genaueres siehe andere Config Dateien, z.B.: poky/meta-poky/conf/distro/poky.conf

Im nächsten Schritt verwenden wir diesen Layer um einen Package Manager zu  
integrieren.

## Package Manager installieren


Hierzu zuerst die mackerel.conf Datei erneut öffnen und folgendes hinzufügen
```
PACKAGE_CLASSES ?= "package_ipk"
```

Nun zurück zum build-rpi Verzeichnis
```
source poky/oe-init-build-env build-rpi
```

und dort die conf/local.conf anpassen  
Hier muss 'PACKAGE_CLASSES' auskommentiert werden, da wir dies im Distro Layer  
gesetzt haben  
'package-management' aktiviert das Packetmgmt in der Runtime
```
#PACKAGE_CLASSES ?= "package_rpm"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks ssh-server-openssh package-management"
DISTRO = "mackerel"
```

Nun das Image erneut erstellen und auf die SD-Karte flashen
```
bitbake -c clean rpi-test-image
bitbake rpi-test-image
```

Dieses mal liegt das fertige Image allerdings in einem anderem Verzeichnis.  
Nun das Image erneut auf die SD Karte flashen.
```
tmp-glibc/deploy/images/raspberrypi4-64/rpi-testimage*wic.bz2
```

Sobald der Pi gebootet hat sollte nun erneut der SSH Zugriff funktionieren.  
Da sich der Fingerprint vom Pi geändert hat wird das Hostsystem eine  
Fehlermeldung anzeigen, dass eine Impersonation vorliegt.
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
```

Dies ist normal und kann mit folgendem Befehl behoben werden.  
Hiermit werden die abgespeicherten Fingerprints in der 'known_hosts' Datei  
gelöscht.
```
ssh-keygen -f "/home/stephan/.ssh/known_hosts" -R "raspberrypi4-64"
```
Anschließend kann man sich erneut mit dem Pi verbinden
```
ssh root@raspberrpi4-64
```

Zudem sollte nun der Package Manager installiert sein, dies kann man wie folgt  
prüfen
```
$ which opkg
/usr/bin/opkg
```

## Remote Package Management Server konfigurieren
Wie gehabt wieder ins 'build-rpi' Verzeichnis und die benötigten Variablen  
setzten lassen
```
source poky/oe-init-build-env build-rpi
```

Nun muss zuerst unser Test Package: curl gebaut und der index aufgesetzt werden
```
bitbake curl
bitbake package-index
```

Daraufhin sollten im Verzeichnis 'tmp-glibc/deploy/ipk' drei Order erstellt  
worden sein
```
├── all
├── cortexa72
└── raspberrypi4_64
```

Nun ins besagte Verzeichnis wechseln und einen HTTP Package Server starten.  
(IP Adresse zur Host IP austauschen)
```
cd tmp-glibc/deploy/ipk
sudo python3 -m http.server --bind 192.168.178.74 80
```


## opkg Client konfigurieren
Nun auf dem Client (dem Pi) die opkg.conf Datei (/etc/opkg/opkg.conf) wie folgt  
anpassen.  
Natürlich wieder mit der passenden IP!
```
# Must have one or more source entries of the form:
#
#   src <src-name> <source-url>
#
# and one or more destination entries of the form:
#
#   dest <dest-name> <target-path>
#
# where <src-name> and <dest-names> are identifiers that
# should match [a-zA-Z0-9._-]+, <source-url> should be a
# URL that points to a directory containing a Familiar
# Packages file, and <target-path> should be a directory
# that exists on the target system.

src/gz all http://192.168.178.74/all
src/gz cortexa72 http://192.168.178.74/cortexa72
src/gz raspberrypi4_64 http://192.168.178.74/raspberrypi4_64

# Proxy Support
#option http_proxy http://proxy.tld:3128
#option ftp_proxy http://proxy.tld:3128
#option proxy_username <username>
#option proxy_password <password>

# Enable GPGME signature
# option check_signature 1

# Offline mode (for use in constructing flash images offline)
#option offline_root target

# Default destination for installed packages
dest root /
option lists_dir   /var/lib/opkg/lists
option info_dir    /var/lib/opkg/info
option status_file /var/lib/opkg/status
```

## opkg verwenden
Anschließend kann der Pi auf den HTTP Server auf dem Host zugreifen.  
Dies kann geprüft werden indem auf dem Pi ein update durchgeführt wird:
```
$ opkg update

Downloading http://192.168.1.69/all/Packages.gz.
Updated source 'all'.
Downloading http://192.168.1.69/aarch64/Packages.gz.
Updated source 'aarch64'.
Downloading http://192.168.1.69/raspberrypi4_64/Packages.gz.
Updated source 'raspberrypi4_64'.
```

Zeitgleich wird auf dem Host in der Shell der Zugriff angezeigt:
```
192.168.178.100 - - [08/May/2024 11:12:38] "GET /all/Packages.gz HTTP/1.1" 200 -
192.168.178.100 - - [08/May/2024 11:12:38] "GET /cortexa72/Packages.gz HTTP/1.1" 200 -
192.168.178.100 - - [08/May/2024 11:12:38] "GET /raspberrypi4_64/Packages.gz HTTP/1.1" 200 -
```

Aber jetzt installieren wir doch mal ein neues Package auf dem Pi.

Wie wir mit diesem Befehl sehen können, ist 'curl' aktuell nicht installiert,  
da kein Pfad der Executable ausgegeben wird.
```
$ which curl

# Alternative kann die Executable einfach ausgeführt werden
$ curl
-sh: curl: not found
```

Nun kann das zuvor gebaute 'curl' Package auf dem Pi vom Host heruntergeladen  
und installiert werden.
```
$ opkg install curl

Installing libcurl4 (7.82.0) on root
Downloading http://192.168.178.74/cortexa72/libcurl4_7.82.0-r0_cortexa72.ipk.
Installing curl (7.82.0) on root
Downloading http://192.168.178.74/cortexa72/curl_7.82.0-r0_cortexa72.ipk.
Configuring libcurl4.
Configuring curl.
```

Wie bereits zuvor wird ebenfalls der Zugriff in der Host Shell angezeigt:
```
192.168.178.100 - - [08/May/2024 11:17:37] "GET /cortexa72/libcurl4_7.82.0-r0_cortexa72.ipk HTTP/1.1" 200 -
192.168.178.100 - - [08/May/2024 11:17:37] "GET /cortexa72/curl_7.82.0-r0_cortexa72.ipk HTTP/1.1" 200 -
```

Anschließend kann erneut mit 'which' geprüft werden, ob es nun installiert ist
```
$ which curl
/usr/bin/curl
```

Sofern 'Upgrades' auf dem Remote Package Maanagement Server zur Verfügung  
stehen, können diese wie gehabt installiert werden
```
opkg update
opkg upgrade
```

Zudem können hiermit die Upgrades vor Installation aufgelistet werden
```
opkg list-upgradable
```