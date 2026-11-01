## Notes install hôte KVM pour PCI passthrough

Fait le 2026-10-28 sur PC fixe modeste doté d'une carte NVIDIA 1060 3Gb VRAM

Localement sur le PC fixe après install fraiche de Debian 13 avant ou sans environnement de bureau:
```sh
apt update
apt install ssh vim
update-alternative --config editor

# provisionner un /home/ludolpif/.ssh/authorized_keys
# tester depuis mon laptop puis:
editor /etc/ssh/sshd_config
# PasswordAuthentication no

editor /etc/apt/sources.list
apt update
apt install nvidia-driver linux-headers-amd64 
# driver nouveau auto blacklisté, l'initrd est regénéré en conséquence
reboot

# Aller dans le BIOS vérifier qu'on a activé le IOMMU
# Vérifier que le kernel a pu l'avoir après boot avec:
dmesg | grep -E 'DMAR|IOMMU'
# Vérfier qu'aucun service n'est HS avant la suite
systemctl --failed

apt install virt-manager
adduser ludolpif libvirt

lspci -nnv | grep NVIDIA
# 08:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1) (prog-if 00 [VGA controller])
# 08:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1) (prog-if 00 [HDA compatible])

echo 'options vfio-pci ids=10de:1c02,10de:10f1' > /etc/modprobe.d/vfio.conf
mv /etc/modprobe.d/nvidia.conf /etc/modprobe.d/nvidia.disabled
systemctl disable nvidia-persistenced.service 
# driver nvidia inhibé, il n'y aura plus d'affichage par l'hôte

# Optionnel, au choix
systemctl disable gdm3
systemctl disable ssdm

cat >> /etc/initramfs-tools/modules <<EOT
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
EOT
update-initramfs -u -k all

# le driver vfio se réservera la carte NVIDIA
# note: si nouveau n'est pas blacklisté, les 2 se marchent dessus
reboot
```

Depuis mon laptop, en tant que root:
```sh
apt install virt-manager --no-install-recommends
apt install gir1.2-spiceclientgtk-3.0
```

Depuis mon laptop, en tant que ludolpif:
```sh
ssh-add -l
ssh-add # charger la clé par défaut
ssh ludolpif@192.168.1.110 run0 systemctl --failed
ssh ludolpif@192.168.1.110 lsmod | grep -E '(nouveau|nvidia|vfio)'
# toujours sur le laptop:
virt-manager
```
- Ajouter une connexion distante ssh `ludolpif@192.168.1.110`
- Créer une VM Windows ou Linux
- y ajouter 2 périphériques PCI listés avant (`08:00.*` dans mon cas)
- y ajouter 2 périphériques USB (claviers et souris physiques)
- Lancer la VM, elle s'affiche sur les écrans physiques de la machine hôte
- Installer le driver NVIDIA dans la VM
- Lancer GPU-Z et un jeu vidéo
- Possibilité de faire du bureau étendu entre la NVIDIA physique et un affichage Spice potentiellement distant

