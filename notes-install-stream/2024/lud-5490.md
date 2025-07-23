Pense bête

- Alt+F2 gted à la place de gedit 
- LXQt utilise Xorg par défaut

Install debian 12 2023-02-26 sur lud-5490 pour streams

- Install from iso (alpha2 sorti avant hier)
- user : ludolpif, mdp root et user settés (ludolpif n'est pas pas sudoer)
- Tout par défaut, dans tasksel choisir LxQT et ssh
- Premier boot = crash car fsck.ext4 1.17 est nécessaire
- (recopié depuis le live : /sbin/e2fsck et /lib/x86_64-linux-gnu/libext2fs.so.2.4 puis update-initramfs qui passe en zstd)
- premier login, Choisir Kwin comme gestionnaire de fenêtres
- popup gestion de l'énergie, faire la config (Paramètres de grestion de l'alimentation)
  - Batterie : niveau faible : 10%
  - Capot : Sur batterie : suspendre, branché : éteindre moniteur
  - Inactivité : décocher "configurer le comportement en cas d'inactivité"

Rendre systemd bloquant moins longtemps si quelque chose déraille

root@lud-5490:~# vim /etc/systemd/system.conf
DefaultTimeoutStartSec=9s
DefaultTimeoutStopSec=9s
#DefaultTimeoutAbortSec=
DefaultDeviceTimeoutSec=9s

Désactiver quelques périphériques (car les fallbacks et defaut dans obs, easyeffects, etc ça déconne)

root@lud-5490:~# cat > /etc/udev/rules.d/90-blacklist-unused-audio-video-capture-dev.rules <<"EOT"
# lsusb 
# udevadm info -a -p /sys/class/video4linux/video3
# udevadm control --reload-rules # for devices than can be unplugged then replugged

# Disables Logitech, Inc. Webcam C930e audio
SUBSYSTEM=="usb", DRIVER=="snd-usb-audio", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="0843", ATTR{authorized}="0"
# Disable 0c45:6717 Microdia Integrated_Webcam_HD audio and video
SUBSYSTEM=="usb", ATTRS{idVendor}=="0c45", ATTRS{idProduct}=="6717", ATTR{authorized}="0"
EOT

Rendre les ports HDMI disponibles dans pipewire pour ma carte Intel HDA

ArchLinux m'a sauvé, encore une fois https://wiki.archlinux.org/title/WirePlumber

```
# cat > /usr/share/alsa-card-profile/mixer/profile-sets/multiple.conf <<"EOT"
[General]
auto-profiles = no

[Mapping analog-stereo]
device-strings = front:%f
channel-map = left,right
paths-output = analog-output analog-output-lineout analog-output-speaker analog-output-headphones analog-output-headphones-2
paths-input = analog-input-front-mic analog-input-rear-mic analog-input-internal-mic analog-input-dock-mic analog-input analog-input-mic analog-input-linein analog-input-aux analog-input-video analog-input-tvtuner analog-input-fm analog-input-mic-line analog-input-headphone-mic analog-input-headset-mic
priority = 15

[Mapping hdmi-stereo]
description = Digital Stereo (HDMI)
device-strings = hdmi:%f
paths-output = hdmi-output-0
channel-map = left,right
priority = 9
direction = output

[Profile multiple]
description = Analog Stereo Duplex + Digital Stereo (HDMI) Output
output-mappings = analog-stereo hdmi-stereo
input-mappings = analog-stereo
EOT

# cat > /etc/wireplumber/main.lua.d/51-alsa-custom.lua <<"EOT
rule = {
  matches = {
    {
      { "device.nick", "matches", "HDA Intel PCH" },
    },
  },
  apply_properties = {
    ["api.alsa.use-acp"] = true,
    ["api.acp.auto-profile"] = false,
    ["api.acp.auto-port"] = false,
    ["device.profile-set"] = "multiple.conf",
    ["device.profile"] = "multiple",
  },
}
table.insert(alsa_monitor.rules,rule)
EOT
```

Config environnement de bureau
```
root@lud-5490:~# apt install liblxqt-backlight-helper # sinon pas de controle backlight
root@lud-5490:~# apt install ffmpeg # juste set as manual pour éviter un auto-remove après
# Alléger le bureau, virer l'écran de veille
root@lud-5490:~# apt autoremove --purge hv3 ksystemstats thunderbird meteo-qt partitionmanager smplayer synaptic system-config-printer-applet xscreensaver
root@lud-5490:~# ln -s /bin/true /usr/local/bin/xdg-screensaver
# Limiter les démons et les risques de leaks
root@lud-5490:~# apt autoremove --purge clipit diodon geoclue-2.0
# (risque d'install de qlipper par dépendance, disabled by default AFAIK)
root@lud-5490:~# apt autoremove --purge modemmanager # un démon de moins, mais si secours 4G un jour ?
root@lud-5490:~# apt autoremove --purge plymouth # spalsh screen de boot qui masque des détails utiles les jours de panne
```

Bug connu pour le panel sur le mauvais écran, car le "primary screen" n'est plus sauvé comme il faut
https://github.com/lxqt/lxqt-config/issues/925#issuecomment-1465060151


- Menu démarrer / Préférences / LXQt Paramétrage du système / Centre de configuration LxQT
  - Apparence / Style des widgets : Breeze, Charger Palette : Dark
  - Apparence / Style des widgets : Breeze
  - Apparence / Theme d'icône : Breeze Dark
  - Apparence / Curseurs : Breeze
  - Apparence / Style GTK 2 et 3 : Breeze-dark 
TODO : lire un tuto LXQt thème sombre ya ya des pb avec les app QT6 et certains "non dark" mixés
  - Clavier / Disposition du clavier : French (alt.)
  - Paramétreur de session LXQt / Lancement automatique de LxQt : désactiver Qlipper et XScreenSaver
  - Touches de raccourcis
    - Raccourci : Ctrl+Alt+T
    - Description : Terminal
    - Commande : x-terminal-emulator 
  - Clic-droit sur l'heure en bas / Configure "Horloge virtuelle"
    - Cocher Date, format ISO 8601
```
root@lud-5490:~# update-alternatives --config x-terminal-emulator 
# Choisir qterminal
```
  - Préférences de qterminal / Apparence / transparence : 0%
  - Préférences de l'explorateur de fichiers
    - il s'appelle pcmanfm-qt
    - Éditer / Préférences / Avancé / Émulateur de terminal : qterminal
    - Éditer / Préférences / Avancé / Intégration de l'archiveur : xarchiver
  - Préférence bash
```
vim ~/.bashrc
# Pour éviter des leaks en stream :
HISTFILESIZE=0
```

- Installer quelques premiers paquets
```
root@lud-5490:~# apt autoremove --purge qlipper 'libreoffice*' xsane 'cups*' kdeconnect zutty
root@lud-5490:~# apt autoremove --purge fonts-noto-cjk fonts-noto-cjk-extra fonts-noto-ui-extra fonts-noto-extra fonts-noto-core fonts-noto-ui-core fonts-noto-unhinted
root@lud-5490:~# apt install curl vim vlc obs-studio gimp uvcdynctrl
root@lud-5490:~# tee /etc/skel/.vimrc > /root/.vimrc <<EOT
runtime! defaults.vim
runtime! debian.vim
if &diff
  syntax off
else
  set mouse=
end
set background=dark
EOT
ludolpif@lud-5490:~$ cp /etc/skel/.vimrc ~
```

Contournement bug low FPS VLC

TODO: chercher pourquoi avec intel-media-va-driver-non-free:amd64 vlc -v sort beaucoup de message late video frame.
Contournement :
- vlc / Outils / Préférences / Entrées Codecs / Décodage matériel : désactiver

Config environnement réseau

```
root@lud-5490:~# vim /etc/hosts
# ajouter lud-nuc1, lud-gb1

# https://unix.stackexchange.com/questions/487640/disable-wifi-on-connection-to-ethernet-with-networkmanager
root@lud-5490:~# cat > /etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh <<"EOT"
#!/bin/sh
syslog_tag="wifi-wired-exclusive"
interface="$1"
iface_mode="$2"
iface_type=$(nmcli dev | grep "$interface" | tr -s ' ' | cut -d' ' -f2)
iface_state=$(nmcli dev | grep "$interface" | tr -s ' ' | cut -d' ' -f3)

logger -i -t "$syslog_tag" "Interface: $interface = $iface_state ($iface_type) is $iface_mode"

enable_wifi() {
    logger -i -t "$syslog_tag" "Interface $interface ($iface_type) is down, enabling wifi ..."
    nmcli radio wifi on
}

disable_wifi() {
    logger -i -t "$syslog_tag" "Disabling wifi, ethernet connection detected."
    nmcli radio wifi off
}

if [ "$iface_type" = "ethernet" ] && [ "$iface_mode" = "down" ]; then
    enable_wifi
elif [ "$iface_type" = "ethernet" ] && [ "$iface_mode" = "up"  ] && [ "$iface_state" = "connected" ]; then
    disable_wifi
fi
EOT
chmod +x /etc/NetworkManager/dispatcher.d/70-wifi-wired-exclusive.sh
```

SSH via clé uniquement

```
root@lud-gb1:~# apt install ssh rsync
root@lud-gb1:~# cp -a ~ludolpif/.ssh/authorized_keys .ssh/
root@lud-gb1:~# vim /etc/ssh/sshd_config
# PasswordAuthentication no
root@lud-gb1:~# service ssh restart
```

Installer drivers propriétaires pour encoding MP4 (finalement le x264 en software c'est peut-être mieux !)
```
root@lud-gb1:~# vim /etc/apt/sources.list
# ajouter contrib non-free 4 fois
root@lud-gb1:~# apt update
root@lud-gb1:~# apt install intel-media-va-driver-non-free
```

Notes essais systemd-nspawn

D'après https://wiki.archlinux.org/title/systemd-nspawn
```
root@lud-5490:~# apt install systemd-container debootstrap
root@lud-5490:~# debootstrap testing /home/chroot
root@lud-5490:~# systemd-nspawn -D /home/chroot
root@lud-5490:~# echo chroot > /home/chroot/etc/debian_chroot
root@lud-5490:~# vimdiff {/,/home/chroot/}etc/resolv.conf 
root@lud-5490:~# vimdiff {/,/home/chroot/}etc/passwd
root@lud-5490:~# vimdiff {/,/home/chroot/}etc/group
root@lud-5490:~# vimdiff {/,/home/chroot/}etc/shadow
root@lud-5490:~# vimdiff {/,/home/chroot/}etc/gshadow
root@lud-5490:~# cp -r /etc/skel /home/chroot/home/ludolpif
root@lud-5490:~# chown -R ludolpif: /home/chroot/home/ludolpif
root@lud-5490:~# mv ~ludolpif/Téléchargements/rustup-init.sh /home/chroot/home/ludolpif/
root@lud-5490:~# systemd-nspawn -bD /home/chroot
Spawning container chroot on /home/chroot.
Press ^] three times within 1s to kill container.
[...]
Welcome to Debian GNU/Linux 12 (bookworm)!
[...]
lud-5490 login: ludolpif
Password: 
(chroot)ludolpif@lud-5490:~$ su -
(chroot)ludolpif@lud-5490:~# apt install npm tree

# ASCIINema player
(chroot)ludolpif@lud-5490:~$ git clone https://github.com/asciinema/asciinema-player
(chroot)ludolpif@lud-5490:~$ grep -A 10 -B2 rustup asciinema-player README.md
(chroot)ludolpif@lud-5490:~$ ./rustup-init.sh
(chroot)ludolpif@lud-5490:~$ cd asciinema-player
(chroot)ludolpif@lud-5490:~/asciinema-player$ git submodule update --init
(chroot)ludolpif@lud-5490:~/asciinema-player$ rustup target add wasm32-unknown-unknown
(chroot)ludolpif@lud-5490:~/asciinema-player$ npm install
(chroot)ludolpif@lud-5490:~/asciinema-player$ npm run build
(chroot)ludolpif@lud-5490:~/asciinema-player$ npm run bundle
(chroot)ludolpif@lud-5490:~/asciinema-player$ tree dist/
dist/
|-- bundle
|   |-- asciinema-player.css
|   |-- asciinema-player.js
|   `-- asciinema-player.min.js
`-- index.js

2 directories, 4 files
(chroot)ludolpif@lud-5490:~/asciinema-player$ git remote myfork git@github.com:ludolpif/asciinema-player.git
# Some PR for improovements
```
Installation NDI5 + obs-ndi

En espérant que les bugs de son manglé à droite sont définitivement des problèmes du passé.
https://github.com/obs-ndi/obs-ndi/releases 

```
ludolpif@lud-5490:~$ mkdir -p obs/bin
ludolpif@lud-5490:~$ cd obs/bin
# Marche pas dans debian 12 (wrong libs / ABI obs)
# ludolpif@lud-5490:~/obs/bin$ wget https://github.com/obs-ndi/obs-ndi/releases/download/4.11.1/obs-ndi-4.11.1-linux-x86_64.deb

ludolpif@lud-5490:~/obs/bin$ wget https://github.com/obs-ndi/obs-ndi/releases/download/4.11.1/libndi5_5.5.3-1_amd64.deb
root@lud-5490:~# dpkg -i ~ludolpif/obs/bin/libndi5*.deb

Alternative essayée aussi (car coredump du plugin) mais non conservée en prod :
# root@lud-5490:/opt# editor ./ndi5-runtime-installer.sh
# Prendre le snippet fourni dans le README, enlever les sudo

Le plugin obs-ndi binaire n'est pas compilé pour les même sversions de QT que obs, donc compilation à la main

root@lud-5490:/# chroot /home/chroot bash
(chroot)root@lud-5490:/# adduser ludolpif sudo
(chroot)root@lud-5490:/# apt install sudo
(chroot)root@lud-5490:/# su - ludolpif
(chroot)ludolpif@lud-5490:~$ git clone https://github.com/obs-ndi/obs-ndi.git
(chroot)ludolpif@lud-5490:~$ cd obs-ndi
(chroot)ludolpif@lud-5490:~/obs-ndi$ .github/scripts/build-linux.sh
[...]
(chroot)ludolpif@lud-5490:~/obs-ndi$ exit
(chroot)root@lud-5490:/# exit

ludolpif@lud-5490:~/obs/bin/plugins/obs-ndi/my-build$ cp -a /home/chroot/home/ludolpif/obs-ndi/release/obs-plugins/64bit/obs-ndi.so .
ludolpif@lud-5490:~/obs/bin/plugins/obs-ndi/my-build$ cp -ar /home/chroot/home/ludolpif/obs-ndi/release/data/obs-plugins/obs-ndi/locale .


root@lud-5490:/opt# mkdir -p /usr/local/lib/x86_64-linux-gnu/obs-plugins/
root@lud-5490:/opt# cp ~ludolpif/obs/bin/plugins/obs-ndi/my-build/obs-ndi.so /usr/local/lib/x86_64-linux-gnu/obs-plugins/
root@lud-5490:/opt# mkdir -p /usr/local/share/obs/obs-plugins/
root@lud-5490:/opt# cp -r ~ludolpif/obs/bin/plugins/obs-ndi/my-build/locale /usr/local/share/obs/obs-plugins/

Les chemins dans /usr/local ne sont pas utilisés par OBS apparement (différence packaging debian vs .deb fait par l'équipe OBS)...
root@lud-5490:/opt# ln -s /usr/local/lib/x86_64-linux-gnu/obs-plugins/obs-ndi.so /usr/lib/x86_64-linux-gnu/obs-plugins/
root@lud-5490:/opt# ln -s /usr/local/share/obs/obs-plugins /usr/share/obs/obs-plugins/obs-ndi


root@lud-5490:/home/chroot# mount -t devtmpfs none dev
root@lud-5490:/home/chroot# mount -t devpts none dev/pts
root@lud-5490:/home/chroot# mount -t proc proc proc
root@lud-5490:/home/chroot# mount -t sysfs sysfs sys
root@lud-5490:/home/chroot# chroot /home/chroot/ bash
(chroot)root@lud-5490:/# apt update
(chroot)root@lud-5490:/# apt upgrade
(chroot)root@lud-5490:/# apt build-dep obs-studio
(chroot)root@lud-5490:/# su - ludolpif
(chroot)ludolpif@lud-5490:~$ git clone https://github.com/obsproject/obs-studio.git
(chroot)ludolpif@lud-5490:~$ git checkout 29.1.3
(chroot)ludolpif@lud-5490:~$ cd obs-studio
(chroot)ludolpif@lud-5490:~$ CI/build-linux.sh --help
(chroot)ludolpif@lud-5490:~/obs-studio$ CI/build-linux.sh
  + Fetching OBS tags...
[OBS-Studio] Set up apt
[sudo] password for ludolpif: 
[...]

(chroot)ludolpif@lud-5490:~$ cmake --build build --target package
exit
ludolpif@lud-5490:~$ cd obs/bin
ludolpif@lud-5490:~/obs/bin$ cp -a /home/chroot/home/ludolpif/obs-studio/build/obs-studio-29.1.3-17-Linux.deb .
ludolpif@lud-5490:~/obs/bin$ su -
root@lud-5490:~# dpkg -i ~ludolpif/obs/bin/obs-studio-29.1.1-17-Linux.deb
(Lecture de la base de données... 290639 fichiers et répertoires déjà installés.)
Préparation du dépaquetage de .../obs-studio-29.1.1-17-Linux.deb ...
Dépaquetage de obs-studio (29.1.3-17) sur (29.1.0-12) ...
Paramétrage de obs-studio (29.1.3-17) ...
root@lud-5490:~# 
ludolpif@lud-5490:~$ obs --verbose
```

Bug report https://github.com/obsproject/obs-studio/issues/8946



Notes essais pipewire + easyeffects
```
root@lud-gb1:~# apt install --purge easyeffects qpwgraph pipewire-audio-client-libraries libspa-0.2-jack
root@lud-gb1:~# cp /usr/share/doc/pipewire/examples/ld.so.conf.d/pipewire-jack-*.conf /etc/ld.so.conf.d/
root@lud-gb1:~# ldconfig
root@lud-gb1:~# reboot

ludolpif@lud-gb1:~$ pw-link -o # outputs, -i input et -l links
ludolpif@lud-gb1:~$ qpwgraph
# recommandé ici : https://wiki.debian.org/PipeWire#Debian_Testing.2FUnstable
TODO ? root@lud-gb1:~# apt autoremove --purge pavucontrol-qt


Config easyeffects

- Préférences / Lancer le service au démarrage : oui
- Préférences / Traiter tous les flux (d'entrée|de sortie) : non
- Préférences / Gestion des périphériques : entrée et sortie : UA-25EX 
  - pour éviter les boucles avec OBS monitor sur la default
- Préférences / Style / thème sombre : oui
- Sortie / Ajouter un effet
  - Porte
    - Seuil : -45,0 dB # Was -52,0 mais mange pas mal le son clavier
  - Speech Processor 
    - Denoise : oui
    - Automatic Gain Control : oui
    - Noise Suppression : -9,0 dB
    - Entrée : 0,0 dB # Essayé 6,0 dB mais ça skip le compresseur HW de la carte son si je m'exclame
    - Sortie : 9,0 dB
  - Retard # Ne laisser que pour le debug, même "en veille" il persiste
    - Gauche 1000,0 ms
    - En veille sauf pour debug
# Remarque : sauvegarder un profil avec le même nom qu'un profil existant ne foncitonne pas, il faut supprimer et recréer pour sauver

Premier lancement OBS
- Alt-F2 obs
- Assistant de config : Annuler
- Fichier / Paramètres / Général / Sortie
  - Confirmation à l'arrêt d'un stream
  - Enregistrement auto lors d'un stream
- Fichier / Paramètres / Général / Déclanchement d'alignement des sources
  - Décocher "Déclancher avec le bord de l'écran # pour placer la camera librement
- Sortie / Streaming
  - Mode de Sortie : Avancé
  - Encodeur : FFMpeg VAAPI H.264 (NVENC)
  - Appareil VAAPI : UHD Graphics 620
  - Profil : High
  - Débit vidéo : 6000kbps
  - Image-clés : 2s
- Sortie / Enregistrement / Chemin : /home/ludolpif/Vidéos 
- Audio / Périphériques audio globaux
  - Audio du bureau : Default (on passera le son du micro dedans via easyeffects+qpwgraph)
  - le reste : Désactivé
- Vidéo / Résolution de sortie : 1536*864, 60fps
- Avancé / Enregistrement / Format nom : %CCYY-%MM-%DD_%hh-%mm-%ss-stream-%VF-%ORES-%FPSfps

Config KeepassXC
```
root@lud-5490:~# apt install keepassxc
```

Config navigateurs

Firefox : général purpose (hors twitch.tv)

- Préférences / Accueil
  - Accueil / Contenu de la page d'accueil : Raccourcis, non
  - Ctrl+Shift+B pour hide la bookmarks bar
- Préférences / Vie privée
  - Cookie et données de sites : Supprimer à la fermeture de Firefox
  - Identifiants et mots de passe : non
  - Historique : Vider l'historique à la fermeture de Firefox

Chromium : pour le dashboard + modo tchat Twitch (car le ReactJS est très mal optimisé avec Firefox)
```
root@lud-5490:~# apt install chromium
```
- Settings / On startup	/ Specifiet set : https://dashboard.twitch.tv/u/ludolpif/stream-manager

Installation Discord
```
root@lud-5490:~# cd ~ludolpif/Téléchargements/
root@lud-5490:/home/ludolpif/Téléchargements# dpkg -i discord-0.0.28.deb 
root@lud-5490:/home/ludolpif/Téléchargements# apt install -f
root@lud-5490:/home/ludolpif/Téléchargements# exit
ludolpif@lud-5490:~$ discord
# penser a exlcure ~/.config/discord/*Cache des backups
```

Installation module v4l2loopback pour camera virtuelle dans OBS
```
root@lud-5490:~# apt install v4l2loopback-utils
# attention ça amène dkms et 450 Mio de toolchain / headers
# modprobe v4l2loopback nécessite soit une signature préalable pour secure boot, soit pas de secureboot
# Supprimé le 2024-04-18 en attente d'usage réel
```

Installation de shadertastic 0.0.3

Release 0.0.3 pre-compilée pour OBS 29 linux 64 bits fournie par xurei
```
root@lud-5490:~# cp /tmp/bin/64bit/shadertastic.so /usr/local/lib/obs-plugins
root@lud-5490:~# cp -r /tmp/data /usr/local/share/obs/obs-plugins/shadertastic

root@lud-5490:~# cd /usr/local/lib/obs-plugins/
root@lud-5490:/usr/local/lib/obs-plugins# ls -al {obs-filters,shadertastic}.so
-rw-r--r-- 1 root root  240808  3 juil. 20:09 obs-filters.so
-rwxr-xr-x 1 root root 1471000 21 août  11:51 shadertastic.so
root@lud-5490:/usr/local/lib/obs-plugins# chmod 644 shadertastic.so 

root@lud-5490:~# cd /usr/local/share/obs/obs-plugins
root@lud-5490:/usr/local/share/obs/obs-plugins# ls -ld {obs-filters,shadertastic}/locale
drwxr-xr-x 2 root root 12288  3 juil. 20:11 obs-filters/locale
drwxr-xr-x 2 root root  4096 21 août  11:56 shadertastic/locale
root@lud-5490:/usr/local/share/obs/obs-plugins# chown -R ludolpif: shadertastic/effects/{filters,transitions}

ludolpif@lud-5490:~/obs$ ln -s /usr/local/share/obs/obs-plugins/shadertastic/effects/filters
ludolpif@lud-5490:~/obs$ ln -s /usr/local/share/obs/obs-plugins/shadertastic/effects/transitions/
ludolpif@lud-5490:~/obs$ ln -s ~/.config/obs-studio/logs/
ludolpif@lud-5490:~/obs$ ls
backup  bin  filters  LICENSE  logs  README.md  scripts  transitions  visuals
ludolpif@lud-5490:~/obs$ 
```

Divers logiciels créatifs

```
root@lud-5490:~# apt install asciinema audacity
root@lud-5490:~# apt install kdenlive dvdauthor-
```

Config arduino pour MIDI

```
root@lud-5490:~# adduser ludolpif dialout
root@lud-5490:~# apt install arduino gmidimonitor
# https://learn.sparkfun.com/tutorials/pro-micro--fio-v3-hookup-guide/all

- Fichier / Préférences / URL de gestionnaire des cartes supplémentaires : 
  - https://raw.githubusercontent.com/sparkfun/Arduino_Boards/master/IDE_Board_Manager/package_sparkfun_index.json
- Outils / Type de carte / Gestionnaire de carte, chercher "Sparkfun AVR Boards"
  - Installer la 1.1.13
- Editer ~/.arduino15/packages/SparkFun/hardware/avr/1.1.13/platform.txt

ludolpif@lud-5490:~$ diff -Naur Documents/debug/sparkfun-avt-1.1.13-platform-with-fixed-path.txt .arduino15/packages/SparkFun/hardware/avr/1.1.13/platform.txt 
--- Documents/debug/sparkfun-avt-1.1.13-platform-with-fixed-path.txt    2023-05-19 22:47:44.856951393 +0200
+++ .arduino15/packages/SparkFun/hardware/avr/1.1.13/platform.txt       2019-11-05 20:50:51.000000000 +0100
@@ -18,7 +18,7 @@
 compiler.warning_flags.all=-Wall -Wextra
 
 # Default "compiler.path" is correct, change only if you want to override the initial value
-compiler.path=/usr/bin/
+compiler.path={runtime.tools.avr-gcc.path}/bin/
 compiler.c.cmd=avr-gcc
 compiler.c.flags=-c -g -Os {compiler.warning_flags} -std=gnu11 -ffunction-sections -fdata-sections -MMD -flto -fno-fat-lto-objects
 compiler.c.elf.flags={compiler.warning_flags} -Os -g -flto -fuse-linker-plugin -Wl,--gc-sections
@@ -90,9 +90,9 @@
 
 # AVR Uploader/Programmers tools
 # ------------------------------
-tools.avrdude.path=/usr/share/arduino/hardware/tools
-tools.avrdude.cmd.path={path}/avrdude
-tools.avrdude.config.path={path}/avrdude.conf
+tools.avrdude.path={runtime.tools.avrdude.path}
+tools.avrdude.cmd.path={path}/bin/avrdude
+tools.avrdude.config.path={path}/etc/avrdude.conf
 
 tools.avrdude.network_cmd={runtime.tools.arduinoOTA.path}/bin/arduinoOTA

- Fermer la session (pour le groupe)
- Réouvrir, lancer arduino IDE
- Type de carte : SparkFun Pro Micro
- Processeur : 3.3V, 8Mhz

- Installer la libraries MIDIUSB
  - https://www.arduino.cc/reference/en/libraries/midiusb/
  - Outil / Gérer les bibliothèques / MIDIUSB v1.0.5
- Fichier / Exemple / MIDIUSB / MIDIUSB_read
  - Vérifier *vraiment* qu'on a bien pro Micro 8Mhz (sinon brick)
  - Compiler / Téléverser
- Tester le MIDI
  - Outil / Moniteur Série
  - Dans qpwgraph, relier le clavier Midi à l'entrée Midi de l'arduino

Devrai recevoir des évènements du genre :
Received: 9-90-3C-17
Received: 8-80-3C-0
Received: 9-90-3C-1C
Received: 8-80-3C-0

- En cas de brick :
  - le bootloader ne dure que 750ms après un reset donc pour reflasher
  - pré-compiler un blink
  - il faut doucle-click le reset (il faut le short to GND)
  - du coup le bootload reste 8 secondes
  - téléverser le blink pendnat ce laps de temps là
  - normalement c'est good
```

Config Pianoteq8 / OrganTeq2

Téléchargés avec Firefox : pianoteq_stage_linux_trial_v813.7z et organteq_linux_trial_v203.7z
Décompressés dans /opt

Pour avoir le droit de faire du realtime sans modifier /etc/security/limits.conf, une idée basique :
```
root@lud-5490:~# adduser ludolpif pipewire
root@lud-5490:~# apt install cpufrequtils
root@lud-5490:/opt/organteq2# for p in $(sed -ne 's/^processor\s\+:\s\+//p' /proc/cpuinfo); do cpufreq-set -c $p -g performance; done

```

Config Oscilloscope (PicoScope)

root@lud-5490:~# curl https://labs.picotech.com/Release.gpg.key | gpg --dearmor > /opt/picoscope7.gpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   959  100   959    0     0   6390      0 --:--:-- --:--:-- --:--:--  6436
root@lud-5490:~# file /opt/picoscope7.gpg
/opt/picoscope7.gpg: OpenPGP Public Key Version 4, Created Wed Aug 14 14:11:52 2013, RSA (Encrypt or Sign, 2048 bits); User ID; Signature; OpenPGP Certificate
root@lud-5490:~# echo "deb [signed-by=/opt/picoscope7.gpg] https://labs.picotech.com/picoscope7/debian/ picoscope main" >/etc/apt/sources.list.d/picoscope7.list
root@lud-5490:~# apt update
Réception de :1 http://ftp.fr.debian.org/debian bookworm InRelease [193 kB]
Réception de :2 http://security.debian.org/debian-security bookworm-security InRelease [48,0 kB]     
Réception de :3 https://labs.picotech.com/picoscope7/debian picoscope InRelease [2 886 B]                          
Réception de :4 http://ftp.fr.debian.org/debian bookworm/non-free-firmware Sources.diff/Index [21,8 kB]
[...]
root@lud-5490:~# apt install picoscope

# 500 Mo pour Mono :'(
