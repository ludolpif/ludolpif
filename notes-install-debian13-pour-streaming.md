## Pense bête

- Alt+F2 featherpad à la place de gedit
- LXQt utilise Xorg par défaut

## Installation debian depuis ISO

- Install from iso debian-testing weekly 2025-06-08 (debian 13)
- user : ludolpif, mdp root et user settés (ludolpif n'est pas pas sudoer, mais sudo est installé)
- Tout par défaut, dans tasksel choisir LxQT, LXQT, utilitaires usuels du système
- premier boot, le login manager affiche un gros clavier visuel, layout US mais passe en FR pour tapper le password
- popup gestion de l'énergie, faire la config (Paramètres de grestion de l'alimentation)
  - Batterie : niveau faible : 10%, quand la batterie est faible: ne rien faire
  - Capot : Sur batterie : suspendre, branché : éteindre moniteur
  - Inactivité : décocher "configurer le comportement en cas d'inactivité"

```
root@lud-5490:~# apt install lxqt-panel lxqt-core mpv # pour marquer en manuel
root@lud-5490:~# apt autoremove --purge xsane cups-server-common libreoffice-common thunderbird hexchat meteo-qt synaptic xscreensaver avahi-daemon bluez gcr gnome-accessibility-themes sudo orca plymouth xorg-docs-core
# plymouth == splash screen de boot qui masque des détails utiles les jours de panne
root@lud-5490:~# ln -s /bin/true /usr/local/bin/xdg-screensaver
root@lud-5490:~# rm -r  /etc/cups
```

## Outils que j'aime avoir sous la main
```
root@lud-5490:~# apt install build-essential chromium curl git gmidimonitor gimp hunspell-fr-revised inotify-tools jq keepassxc mate-calc oathtool rsync ssh strace uvcdynctrl vim xarchiver
root@lud-5490:~# sed -i -e 's/^#\?\s*PasswordAuthentication\s\+.*$/PasswordAuthentication no/' /etc/ssh/sshd_config
root@lud-5490:~# service ssh restart
```

## Fix SDDM login screen
```
root@lud-5490:~# apt remove --purge qt6-virtualkeyboard-plugin
```

## Fix systemd
Si un service déconne, le penalty est de 90 secondes par défaut, mettre 9s.

```
root@lud-5490:~# vim /etc/systemd/system.conf
DefaultTimeoutStartSec=9s
DefaultTimeoutStopSec=9s
#DefaultTimeoutAbortSec=
DefaultDeviceTimeoutSec=9s
```

## Enable SysRq to have manual OOM killer

```
root@lud-5490:~# echo "kernel.sysrq = 1" | tee /etc/sysctl.d/sysrq.conf
root@lud-5490:~# /usr/lib/systemd/systemd-sysctl
root@lud-5490:~# cat /proc/sys/kernel/sysrq
1
```

sysrq: HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) memory-full-oom-kill(f) kill-all-tasks(i) thaw-filesystems(j) sak(k) show-backtrace-all-active-cpus(l) show-memory-usage(m) nice-all-RT-tasks(n) poweroff(o) show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) force-fb(v) show-blocked-tasks(w) dump-ftrace-buffer(z) replay-kernel-logs(R)

## Fix brightness keys and 0% luminosity

- Note : `pkexec` est nécessaire pour `/usr/bin/lxqt-config-brightness`, et le paquet liblxqt-backlight-helper semble nécessaire
```
ludolpif@lud-5490:~$ mkdir bin
ludolpif@lud-5490:~$ cat bin/backlight.sh <<EOT
#!/bin/bash
read driver max curr < <(pkexec lxqt-backlight_backend --show)
#echo "driver: $driver, max: $max, curr: $curr"

step_min=$((max/200))
step=$((curr/10))
if [ $step -lt $step_min ]; then step=$step_min; fi
if [ $step -eq 0 ]; then step=1; fi

case $1 in
	--inc) next=$((curr+$step)); if [ $next -gt $max  ]; then next=$max  ; fi ;;
	--dec) next=$((curr-$step)); if [ $next -lt $step ]; then next=$step ; fi ;;
	*) echo "Usage: $0 (--inc|--dec)" >&2; exit 1 ;;
esac
pkexec lxqt-backlight_backend $next
EOT

ludolpif@lud-5490:~$ chmod +x bin/backlight.sh
```

- Dans les raccourcis globaux, associer les touches Backlight à la commande /home/ludolpif/bin/backlight.sh avec --inc et --dec respectivement

## Editeur de texte (featherpad)

Options / Préférences / Texte / Chemin vers le dictionnaire hunspell : `/usr/share/hunspell/fr_FR.dic`

## Terminal

`qterminal` est installé par défaut. Fichier / Préférences... / Apparence / Transparence du terminal : 0%

## Screencapture

- Dans les raccourcis globaux, associer PrintScreen avec rien, shift, alt à screengrab avec les options -r -f -a respectivement.


## Désactiver quelques périphériques (car les fallbacks et defaut dans obs, easyeffects, etc ça déconne)

```
root@lud-5490:~# cat > /etc/udev/rules.d/90-blacklist-unused-audio-video-capture-dev.rules <<"EOT"
# lsusb
# udevadm info -a -p /sys/class/video4linux/video3
# udevadm control --reload-rules # for devices than can be unplugged then replugged

# Disables Logitech, Inc. Webcam C930e audio
SUBSYSTEM=="usb", DRIVER=="snd-usb-audio", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="0843", ATTR{authorized}="0"
# Disable 0c45:6717 Microdia Integrated_Webcam_HD audio and video
SUBSYSTEM=="usb", ATTRS{idVendor}=="0c45", ATTRS{idProduct}=="6717", ATTR{authorized}="0"
EOT
```

- Menu LxQT / icône rouage : Centre de configuration LxQT
  - Clavier / Disposition du clavier : French (alt.)
  - Paramétreur de session LXQt / Lancement automatique de LxQt : désactiver Qlipper et XScreenSaver
  - Touches de raccourcis
    - Raccourci : Ctrl+Alt+T
    - Description : Terminal
    - Commande : x-terminal-emulator
  - Clic-droit sur l'heure en bas / Configurer "Horloge virtuelle"
    - Cocher Date, format ISO 8601, position Avant

## Préférences d'outils ligne de commande


```
root@lud-5490:~# echo 'HISTFILESIZE=0' > /etc/profile.d/no_history.sh

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

## Linux VA driver (avec encodeur HW intel)

Pas ajouté non-free pour uniquement ce paquet.

```
root@lud-5490:/opt# wget http://ftp.fr.debian.org/debian/pool/non-free/i/intel-media-driver-non-free/intel-media-va-driver-non-free_25.2.3+ds1-1_amd64.deb
root@lud-5490:/opt# apt remove --purge intel-media-va-driver
root@lud-5490:/opt# dpkg -i intel-media-va-driver-non-free_25.2.3+ds1-1_amd64.deb
```

## OBS from sources
```
root@lud-5490:~# apt build-dep obs-studio

ludolpif@lud-5490:~$ cd obs/
ludolpif@lud-5490:~/obs$ ls
ludolpif@lud-5490:~/obs$ mkdir pkg
ludolpif@lud-5490:~/obs$ cd pkg/
ludolpif@lud-5490:~/obs/pkg$ apt source obs-studio
ludolpif@lud-5490:~/obs/pkg$ wget https://cdn-fastly.obsproject.com/downloads/cef_binary_5060_linux64.tar.bz2
ludolpif@lud-5490:~/obs/pkg$ tar xf cef_binary_5060_linux64.tar.bz2

ludolpif@lud-5490:~/obs/pkg$ cd obs-studio-*
ludolpif@lud-5490:~/obs/pkg/obs-studio-30.2.3+dfsg$ dpkg-buildpackage -us -uc
# Par défaut, no browser mode, car ne va pas prendre le cef binary car non dispo/facile from sources
# tuto debian 12 : https://obsproject.com/forum/threads/debian-obs-studio-build-mini-howto.169680/
# cef builds https://cef-builds.spotifycdn.com/index.html#linux64
# mais la 30.2.3 dépends toujorus de cef_binary_5060

root@lud-5490:~# dpkg -i ~ludolpif/obs/pkg/{libobs0t64,obs-studio,obs-plugins}_30.2.3+dfsg-3_amd64.deb

# Pour build des plugins
root@lud-5490:~# dpkg -i ~ludolpif/obs/pkg/libobs-dev_30.2.3+dfsg-3_amd64.deb

ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ apt source obs-move-transition
ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ git clone https://github.com/sorayuki/obs-multi-rtmp.git
ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ cd obs-multi-rtmp
ludolpif@lud-5490:~/obs/pkg/try-build-plugins/obs-multi-rtmp$ git checkout a629a19cc62589e9c12f883946c45dca2cc76070 && cd ..
ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ mv obs-multi-rtmp obs-multi-rtmp-0.6.0.1
ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ tar cvzf obs-multi-rtmp_0.6.0.1.orig.tar.gz --exclude .git obs-multi-rtmp-0.6.0.1/
ludolpif@lud-5490:~/obs/pkg/try-build-plugins$ cd obs-multi-rtmp-0.6.0.1/
ludolpif@lud-5490:~/obs/pkg/try-build-plugins/obs-multi-rtmp-0.6.0.1$ tar xf ../obs-move-transition_3.1.2-1.debian.tar.xz && cd debian
ludolpif@lud-5490:~/obs/pkg/try-build-plugins/obs-multi-rtmp-0.6.0.1/debian$ editor *
ludolpif@lud-5490:~/obs/pkg/try-build-plugins/obs-multi-rtmp-0.6.0.1/debian$ cd ..
ludolpif@lud-5490:~/obs/pkg/try-build-plugins/obs-multi-rtmp-0.6.0.1$ dpkg-buildpackage -us -uc

```


## Discord

https://github.com/palfrey/discord-apt?tab=readme-ov-file

```
root@lud-5490:~# dpkg -i ~ludolpif/Téléchargements/discord-repo_1.0_all.deb
root@lud-5490:~# apt install -f
root@lud-5490:~# apt update
root@lud-5490:~# apt install discord

ludolpif@lud-5490:~$ discord
ludolpif@lud-5490:~$ touch ~ludolpif/.config/discord/Cache/NOBACKUPDIR.TAG
```

## git

```
# Import depuis une précédente install (ou regénérer toutes les clés)
ludolpif@lud-5490:/mnt/tmp/home/ludolpif/.ssh$ mv authorized_keys id_rsa* ~/.ssh/

ludolpif@lud-5490:~$ git config --global user.email ludolpif@gmail.com
ludolpif@lud-5490:~$ git config --global user.name ludolpif
ludolpif@lud-5490:~$ mkdir git
ludolpif@lud-5490:~$ cd git
ludolpif@lud-5490:~/git$ git clone https://github.com/ludolpif/ludolpif
ludolpif@lud-5490:~/git$ cd ludolpif
```

## Rust + Bevy

https://www.rust-lang.org/learn/get-started

```
ludolpif@lud-5490:~/git$ mkdir bevy
ludolpif@lud-5490:~/git$ cd bevy/
ludolpif@lud-5490:~/git/bevy$ git clone -b v0.16.1 https://github.com/bevyengine/bevy
ludolpif@lud-5490:~/git/bevy$ git clone https://github.com/ludolpif/hello-bevy
ludolpif@lud-5490:~/git/bevy$ cd hello-bevy

root@lud-5490:/home/ludolpif/git/bevy/hello-bevy# ./build-deps-systemwide.sh

ludolpif@lud-5490:~/git/bevy$ editor Cargo.toml
ludolpif@lud-5490:~/git/bevy/hello-bevy$ ./build-deps-userwide.sh
ludolpif@lud-5490:~/git/bevy/hello-bevy$ cargo update
ludolpif@lud-5490:~/git/bevy/hello-bevy$ cargo build
```

## Swapfile et zswap pour rust-analyser + laptop trop juste en RAM


root@lud-5490:/# dd if=/dev/zero of=/home/swap.img bs=1M count=8k
8192+0 enregistrements lus
8192+0 enregistrements écrits
8589934592 octets (8,6 GB, 8,0 GiB) copiés, 36,4064 s, 236 MB/s
root@lud-5490:/# mkswap /home/swap.img
mkswap: /home/swap.img : droits 0664 non sûrs, corriger avec : chmod 0600 /home/swap.img
Configure l'espace d'échange (swap) en version 1, taille = 8 GiB (8589930496 octets)
pas d'étiquette, UUID=add9da34-2c8c-47e6-bfb2-a24c81b69cdf
root@lud-5490:~# chmod 0600 /home/swap.img
root@lud-5490:/# echo "/home/swap.img       none            swap    defaults        0       0" | tee -a /etc/fstab
root@lud-5490:/# systemctl daemon-reload
root@lud-5490:/# swapon -a
root@lud-5490:/# swapon -s
Nom fichier                             Type            Taille          Utilisé         Priorité
/home/swap.img                          file            1048572         0               -2
root@lud-5490:/#


## VSCode avec rust-analyser

```
ludolpif@lud-5490:/tmp$ curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg

root@lud-5490:/tmp# install -o root -g root -m 644 microsoft.gpg /etc/apt/keyrings/microsoft-archive-keyring.gpg
root@lud-5490:/tmp# echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/microsoft-archive-keyring.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list

root@lud-5490:~# apt update
root@lud-5490:~# apt install code

root@lud-5490:~# echo "fs.inotify.max_user_instances = 16384" | tee /etc/sysctl.d/inotify_vscode.conf
root@lud-5490:~# /usr/lib/systemd/systemd-sysctl
root@lud-5490:~# cat /proc/sys/fs/inotify/max_user_instances
16384



ludolpif@lud-5490:/tmp$ cargo new hello_world
ludolpif@lud-5490:/tmp$ cd new hello_world
ludolpif@lud-5490:/tmp/hello_world$ code .
# installer rust-analyser via https://code.visualstudio.com/docs/languages/rust#_code-navigation
# Menu Run / Open Configuration
# édite le .vscode/launch.json
# Ajouter un LD_LIBRARY_PATH tel que cargo run le ferai
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            [...]
            "cwd": "${workspaceFolder}",
            "env": {
                "LD_LIBRARY_PATH": "/home/ludolpif/git/bevy/hello-bevy/target/debug/deps:/home/ludolpif/git/bevy/hello-bevy/target/debug:/home/ludolpif/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib:/home/ludolpif/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib"
            },
        }
    ]
}

# Limiter les pics de consomation de RAM lorsque VSCode lance rust-analyser
ludolpif@lud-5490:~$ editor ~/.cargo/config.toml
[build]
jobs = 2


ludolpif@lud-5490:/tmp/hello_world$ cargo build
ludolpif@lud-5490:/tmp/hello_world$ cargo run
```

Lire https://code.visualstudio.com/docs/languages/rust#_code-navigation


## Pipewire
```
root@lud-5490:~# apt install qpwgraph wireplumber easyeffects pipewire-alsa pulseaudio-
root@lud-5490:~# adduser ludolpif pipewire
```

- "Paramètres de la sesion LxQT"
  - Ajouter, "qpwgraph --minimized", cocher "Attendre le systray"
  - Décocher "Système de son PulseAudio", et "Qlipper"
  - Sauver la patchbay sous ~/obs/streaming-v1.qpwgraph

## EasyEffects

- Préférences / Lancer le service au démarrage : oui
- Préférences / Traiter tous les flux (d'entrée|de sortie) : non
- Préférences / Gestion des périphériques
  - Entrée : UA-25EX
  - Sortie : Audio interne Stéréo analogique
- Préférences / Style / thème sombre : oui
- Entrée / Ajouter un effet
  - Porte
    - Seuil : -45,0 dB # Was -52,0 mais mange pas mal le son clavier
  - Compresseur
    - Entrée : +12 dB
    - Sortie : 0 dB
- Préréglage / Local / Nom : streaming-v1
- Pipewire / Chargement automatique des réglages / Bouton en bas : Entrée
  - Périphérique : UA-25EX
  - Préréglage : streaming-v1
  - Cliquer sur "+" pour ajouter


## Premier lancement OBS
- Alt-F2 obs
- Assistant de config : Annuler
- Fichier / Paramètres / Général / Sortie
  - Confirmation à l'arrêt d'un stream
  - Enregistrement auto lors d'un stream
- Fichier / Paramètres / Général / Déclanchement d'alignement des sources
  - Décocher "Déclancher avec le bord de l'écran # pour placer la camera librement
- Sortie / Streaming
  - Mode de Sortie : Avancé
  - Encodeur : FFMpeg VAAPI H.264
  - Appareil VAAPI : UHD Graphics 620
  - Profil : High
  - Débit vidéo : 2500kbps
  - Image-clés : 2s
- Sortie / Enregistrement / Chemin : /home/ludolpif/Vidéos
- Audio / Périphériques audio globaux
  - Audio du bureau : Désactivé
  - Mic : EasyEffects Source
  - le reste : Désactivé
- Vidéo / Résolution de sortie : 1920*1080 30fps ( ou 1536*864, 60fps )
- Avancé / Enregistrement / Format nom : %CCYY-%MM-%DD_%hh-%mm-%ss-stream-%VF-%ORES-%FPSfps

## Config navigateurs

### Firefox : général purpose (hors twitch.tv)

- Préférences / Accueil
  - Accueil / Contenu de la page d'accueil : Raccourcis, non
  - Ctrl+Shift+B pour hide la bookmarks bar
- Préférences / Vie privée
  - Cookie et données de sites : Supprimer à la fermeture de Firefox
  - Vie privée / Protection renforcée contre le pistage : Stricte
  - Vie privée / Do not sell + Do not track
  - Identifiants et mots de passe : non
  - Historique : Ne jamais conserver l'historique

### TODO Chromium
Pour le dashboard + modo tchat Twitch (car le ReactJS est très mal optimisé avec Firefox)

- Settings / On startup	/ Specifiet set : https://dashboard.twitch.tv/u/ludolpif/stream-manager


TODO reprendre la lecture de Documents/notes-install-lud-5490-debian-12-pour-streaming.txt à 66% du doc

## Arduino

```
root@lud-5490:~# apt install arduino
root@lud-5490:~# adduser ludolpif dialout
```

- Menu Fichier / Préférences / Paramaètres / URL de gestionnaire de cartes supplémentaires
  - Coller https://raw.githubusercontent.com/sparkfun/Arduino_Boards/main/IDE_Board_Manager/package_sparkfun_index.json
- Menu Outils / Type de Carte / Gestionnaire de Carte
  - Chercher sparkfun
  - Installer SparkFun AVR Boards
- Menu Outils / Type de Carte / Sparkfun / SparkFun Pro Micro
- Corriger les chemins pour les outils d'upload du code sur ces board là
```
ludolpif@lud-5490:~$ editor ~/.arduino15/packages/SparkFun/hardware/avr/1.1.13/platform.txt

compiler.path=/usr/bin/

# AVR Uploader/Programmers tools
# ------------------------------
tools.avrdude.path=/usr/share/arduino/hardware/tools
tools.avrdude.cmd.path={path}/avrdude
tools.avrdude.config.path={path}/avrdude.conf
```

## Picoscope (non)

Plutôt le mettre sur le PC fixe, il wrek les associations de fichiers
