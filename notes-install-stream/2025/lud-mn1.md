## Installation debian depuis ISO

- Install from iso debian-testing weekly 2025-06-08 (debian 13)
- user : ludolpif, mdp root et user settés (ludolpif n'est pas pas sudoer, mais sudo est installé)
- Tout par défaut, dans tasksel choisir KDE, Serveur SSH, utilitaires usuels du système

```
$ ssh-copy-id ...
$ ssh root@lud-mn1 # via la clé ssh

root@lud-mn1:~# editor /etc/ssh/sshd_config
# PasswordAuthentication no
root@lud-mn1:~# rm /etc/ssh/sshd_config.d/50-cloud-init.conf
root@lud-mn1:~# service ssh reload
```
## Fix bad Debian defaults pour umask, OpenSSH et QoS (DSCP)

```
root@lud-mn1:~# chfn -o umask=0022 ludolpif
root@lud-mn1:~# umask 0022

root@lud-mn1:~# echo 'IPQoS af21 cs0' > /etc/ssh/ssh_config.d/ssh_modern_IPQoS.conf
root@lud-mn1:~# echo 'IPQoS af21 cs0' > /etc/ssh/sshd_config.d/ssh_modern_IPQoS.conf
root@lud-mn1:~# service ssh reload
```

## Installation des tous les paquets utiles

```
root@lud-mn1:~# apt install apparmor-utils blender btop curl obs-studio rustup sloccount strace vlc vim
```

## NVIDIA

J'ai une 4060 avec 3GB de VRAM donc ni raytracing, ni driver open-sourcé.

```
root@lud-mn1:~# apt install nvidia-driver linux-headers-amd64
root@lud-mn1:~# reboot
```

## Fix systemd

```
root@lud-mn1:~# editor /etc/systemd/system.conf
[Manager]
DefaultTimeoutStartSec=9s
DefaultTimeoutStopSec=9s
DefaultDeviceTimeoutSec=9s
```

## Fix slow shutdown

https://bbs.archlinux.org/viewtopic.php?pid=2232020#p2232020

```
ludolpif@lud-mn1:~$ systemctl --user edit plasma-kwin_x11.service
[Service]
TimeoutStopSec=3s

Successfully installed edited file '/home/ludolpif/.config/systemd/user/plasma-kwin_x11.service.d/override.conf'.
```

## Fix kernel log flooding
```
root@lud-mn1:~# editor /etc/apparmor.d/Xorg
# lpo log flooding
# profile Xorg /usr/lib/xorg/Xorg flags=(complain,attach_disconnected, complain) {
profile Xorg /usr/lib/xorg/Xorg flags=(unconfined,attach_disconnected, unconfined) {
```

## Config Apparence

- TODO: détailler passage en thème sombre mais conserver le thème debian pour SDDM (bug à chaque reboot)
- Configuration du Système / Applications & Fenêtres / Gestion des fenêtres / Effets de bureau
  - Accessibilité : Coche Animation du clic de la souris
    - Configurer / Configuration de base :
      - bouton gauche : vert, central: rose, droit : jaune
    - Configurer / Configuration avancée :
      - Largeur de ligne : 3 px
      - Rayon de l'anneau : 32px
      - Décocher "Afficher le texte" (c'est à moitié en panne, peut petre à cause de l'effet de bureau d'animation des infobulles)
    - Raccourci pour activer après avoir appliqué les paramètres : Méta + *
  -  Accessibilité : Choisir Loupe
    - Largeur : 640px, Hauteur 360px
  - Apparence : cocher Tracé à la Souris
    - Largeur : 6px
    - Couleur : Violet
  - Sécurité et confidentialité / Verrouillange de l'écran
    - Jamais. Décocher après sortie de veille. Il a le "mauvais" thème en plus (pas le même que SDDM).

- Essayé mais désactivé pour l'instant
  - Apparence : Translucidité
    - Fenêtres inactives et Menus : 90%
    - Autres : 100%

## Config navigateurs

### Firefox : général purpose (hors twitch.tv)

- Préférences / Accueil
  - Accueil / Contenu de la page d'accueil : Raccourcis, non
  - Ctrl+Shift+B pour hide la bookmarks bar
- Préférences / Vie privée
  - Cookie et données de sites : Supprimer à la fermeture de Firefox
  - Vie privée / Protection renforcée contre le pistage : Stricte
  - Vie privée / Do not sell + Do not track
  - Identifiants et mots de passe : non
  - Historique : Ne jamais conserver l'historique
- Extensions
  - Dark Reader

```
root@lud-5490:~# cat > /etc/udev/rules.d/90-blacklist-unused-audio-video-capture-dev.rules <<"EOT"
# lsusb
# udevadm info -a -p /sys/class/video4linux/video3
# udevadm control --reload-rules # for devices than can be unplugged then replugged

# Disables Logitech, Inc. Webcam C250 audio
SUBSYSTEM=="usb", DRIVER=="snd-usb-audio", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="0804", ATTR{authorized}="0"
EOT
```

## Limiter les services permettant de l'autoconf ou découverte

Raison : il y a eu assez de CVE autour de ça et je n'en ai pas strictement besoin.
```
root@lud-mn1:~# systemctl stop cups-browsed
root@lud-mn1:~# systemctl disable cups-browsed

root@lud-mn1:~# systemctl stop cups.service
root@lud-mn1:~# systemctl stop cups.socket
root@lud-mn1:~# systemctl disable cups.service
root@lud-mn1:~# systemctl disable cups.socket

root@lud-mn1:~# systemctl stop avahi-daemon.service
root@lud-mn1:~# systemctl disable --now avahi-daemon.service
root@lud-mn1:~# systemctl disable --now avahi-daemon.socket

root@lud-mn1:~# apt autoremove --purge kdeconnect

# Vérifier que c'est propre avec:
root@lud-mn1:~# ss -tuapen
```

## Désactiver l'historique du presse papier (lib? Klipper)

- Mettre un élément dans le presse papier
- Clic-droit sur l'icone presse papier près de l'heure
- Décocher : Enregistrer l'historique pour toutes les sessions de bureau

## Presse papier partagé

```
ludolpif@lud-mn1:~$ cargo install share-clipboard-rs
# En réalité, pour l'instant j'ai forké et corrigé 3 bugs, donc install manuelle

ludolpif@lud-mn1:~$ editor .profile
# set PATH so it includes cargo's bin if it exists
if [ -d "$HOME/.cargo/bin" ] ; then
    PATH="$HOME/.cargo/bin:$PATH"
fi
```

## Voice-to-text

```
root@lud-mn1:~# apt install pipx xdotool

ludolpif@lud-mn1:~$ mkdir ~/bin ~/.local/bin
ludolpif@lud-mn1:~$ cat >> .profile <<"EOT"
# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi

# set PATH so it includes cargo's bin if it exists
if [ -d "$HOME/.cargo/bin" ] ; then
    PATH="$HOME/.cargo/bin:$PATH"
fi
EOT
ludolpif@lud-mn1:~$ source .profile
ludolpif@lud-mn1:~$ echo $PATH
/home/ludolpif/.cargo/bin:/home/ludolpif/.local/bin:/home/ludolpif/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games

ludolpif@lud-mn1:~$ pipx install vosk
ludolpif@lud-mn1:~/Téléchargements$ wget https://alphacephei.com/kaldi/models/vosk-model-small-fr-0.22.zip
ludolpif@lud-mn1:~/git$ git clone https://github.com/ideasman42/nerd-dictation.git

ludolpif@lud-mn1:~$ cd ~/bin/
ludolpif@lud-mn1:~/bin$ cp -a ~/git/nerd-dictation/nerd-dictation .
ludolpif@lud-mn1:~/bin$ sed -i '1s|.*|#!/home/ludolpif/.local/share/pipx/venvs/vosk/bin/python|' nerd-dictation
ludolpif@lud-mn1:~/bin$ mkdir /home/ludolpif/.config/nerd-dictation
ludolpif@lud-mn1:~/bin$ cd /home/ludolpif/.config/nerd-dictation
ludolpif@lud-mn1:~/.config/nerd-dictation$ unzip ~/Téléchargements/vosk-model-small-fr-0.22.zip 
ludolpif@lud-mn1:~/.config/nerd-dictation$ rm vosk-model-small-fr-0.22.zip 
ludolpif@lud-mn1:~/.config/nerd-dictation$ ln -s vosk-model-small-fr model
ludolpif@lud-mn1:~$ nerd-dictation begin --timeout 1.0 --output STDOUT
```
