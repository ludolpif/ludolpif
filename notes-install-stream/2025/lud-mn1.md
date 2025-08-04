## Pense bête

## Installation debian depuis ISO

- Install from iso debian-testing weekly 2025-06-08 (debian 13)
- user : ludolpif, mdp root et user settés (ludolpif n'est pas pas sudoer, mais sudo est installé)
- Tout par défaut, dans tasksel choisir KDE, utilitaires usuels du système

## Config Apparence

TODO détailler passage en thème sombre mais conserver le thème debian pour SDDM

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
