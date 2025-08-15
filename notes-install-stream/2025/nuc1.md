## Installation debian depuis ISO

- Install from iso debian-testing weekly 2025-06-08 (debian 13)
- user : temp, mdp provisoire pour paramétrérer les clés SSH
- tasksel : pas d'env graphique, serveur SSH, utilitaires usuels du système
- popcon : oui

```
$ ssh-copy-id ...
$ ssh root@nuc1 # via la clé ssh

root@nuc1:~# editor /etc/ssh/sshd_config
# PasswordAuthentication no
root@nuc1:~# service ssh reload

root@nuc1:~# deluser temp
root@nuc1:~# rm -r /home/temp
```

## Installation des tous les paquets utiles

```
# Spécifique à mon environnement
root@nuc1:~# apt install firmware-realtek ntfs-3g
# Outils système généraux
root@nuc1:~# apt install rsync screen secure-delete tree unattended-upgrades unzip vim
# Outils pour débogage réseau/système
root@nuc1:~# apt install iftop iotop iperf iperf3 strace tcpdump
# Hébergement site ludolpif.fr
root@nuc1:~# apt install apache2 certbot make python3-certbot-apache php-fpm php-intl php-yaml
# Réseau pour le homelab
root@nuc1:~# apt install radvd systemd-resolved udhcpd vlan wireguard
# Outils streaming
root@nuc1:~# apt install streamlink
```

## Personnalisation minimale pour moi et pour le stream

```
root@nuc1:~# update-alternatives --config editor
# J'aime choisir /usr/bin/vim.basic

root@nuc1:~# cat > /root/.vimrc <<"EOT"
runtime! defaults.vim
runtime! debian.vim
if &diff
	syntax off
else
	set mouse=
endif
EOT


root@nuc1:~# editor ~/.bashrc
# Décommenter pour avoir de la couleur et des alias
# Importer les sections pour .bash_aliases et bash_completion depuis /etc/skel/.bashrc

root@nuc1:~# editor ~/.bash_aliases
# Ajouter des fonctions *_stream() et des alias pour masquer
#  certaines valeurs dans les résultats de commandes interactives
```

## Configuration du système
```
root@nuc1:~# editor /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Mail "root";
Unattended-Upgrade::Automatic-Reboot "true";
root@nuc1:~# /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";

root@nuc1:~# editor /etc/nftables.conf
# Configuration basique pour l'instant
root@nuc1:~# systemctl restart nftables

root@nuc1:~# systemctl enable nftables.service
Created symlink /etc/systemd/system/sysinit.target.wants/nftables.service → /lib/systemd/system/nftables.service.
root@nuc1:~# systemctl start nftables.service

# Script pour debugguer plus facilement des drop en input
root@nuc1:~# editor /usr/local/sbin/nftables-enable-input-drop-log
root@nuc1:~# chmod +x /usr/local/sbin/nftables-enable-input-drop-log


# Configuration DHCP sans route par défaut pour interface USB, branchée à la box internet
root@nuc1:~# editor /etc/systemd/network/dhcp.network
root@nuc1:~# systemctl enable systemd-networkd
root@nuc1:~# networkctl reload
root@nuc1:~# networkctl reconfigure enx000*********
```

## Configuration de mes (borg) backups

```
root@nuc1:~# alias adduserbkp='adduser --system --firstuid 800 --shell /bin/sh'
root@nuc1:~# adduserbkp --gecos 'Borg Backup user lud-mn1' --home /mnt/bkp/lud-mn1 bbkp-mn1
root@nuc1:~# adduserbkp --gecos 'Borg Backup lud-5490' --home /mnt/bkp/lud-5490 bbkp-5490
root@nuc1:~# adduserbkp --gecos 'Borg Backup piou' --home /mnt/bkp/piou bbkp-piou

root@nuc1:~# dpkg -i /opt/borg-family_0.2-1_all.deb
root@nuc1:~# apt install -f

root@nuc1:/etc/borg-family# editor envvars
root@nuc1:/etc/borg-family# apt clean
root@nuc1:/etc/borg-family# bfrun
The authenticity of host '***********' can't be established.
ED25519 key fingerprint is SHA256:****************************************************
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Remote: Warning: Permanently added '***********' (ED25519) to the list of known hosts.
```

## Outil maison pour agir sur le DynDNS fourni par BookMyName

```
root@nuc1:~# editor /etc/bkddns.conf
root@nuc1:~# touch /etc/bkddns-secrets.conf
root@nuc1:~# chmod 0600 /etc/bkddns-secrets.conf
root@nuc1:~# editor /etc/bkddns-secrets.conf

root@nuc1:~# editor /usr/local/bin/bkddns
root@nuc1:~# chmod +x /usr/local/bin/bkddns

root@nuc1:~# editor /etc/networkd-dispatcher/routable.d/bookmyname-dyndns
```

## Autohébergement site web ludolpif.fr

```
cat > /etc/apache2/sites-available/vh-stream.conf <<"EOT"
<VirtualHost *:80>
	ServerName ludolpif.fr
	# Aliases for letsencrypt challenge (all alternate names)
	ServerAlias www.ludolpif.fr
	ServerAdmin abuse@ludolpif.fr
	DocumentRoot /var/www/stream/www

	RewriteEngine on
	RewriteCond %{REQUEST_URI} !^/.well-known/acme-challenge/
	RewriteRule ^/(.*)         https://ludolpif.fr/$1 [L,R=301]

	ErrorLog ${APACHE_LOG_DIR}/stream/error.log
	CustomLog ${APACHE_LOG_DIR}/stream/access.log combined

	<Directory /var/www/stream/www/.well-known/acme-challenge>
		Options None
		AllowOverride None
		Require all granted
	</Directory>
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
EOT
mkdir -p /var/www/stream/www /var/log/apache2/stream/

root@nuc1:~# a2enmod proxy_fcgi rewrite
root@nuc1:~# a2enconf php8.2-fpm
root@nuc1:~# a2ensite vh-stream
root@nuc1:~# a2dissite default-ssl
root@nuc1:~# systemctl restart apache2


root@nuc1:~# certbot --apache
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): ludolpif@gmail.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: n
Account registered.

Which names would you like to activate HTTPS for?
We recommend selecting either all domains, or all domains in a VirtualHost/server block.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: ludolpif.fr
2: www.ludolpif.fr
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
Requesting a certificate for ludolpif.fr and www.ludolpif.fr

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/ludolpif.fr/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/ludolpif.fr/privkey.pem
This certificate expires on 2023-06-17.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Some rewrite rules copied from /etc/apache2/sites-enabled/vh-stream.conf were disabled in the vhost for your
 HTTPS site located at /etc/apache2/sites-available/vh-stream-le-ssl.conf because they have the potential to
 create redirection loops.
Successfully deployed certificate for ludolpif.fr to /etc/apache2/sites-available/vh-stream-le-ssl.conf
Successfully deployed certificate for www.ludolpif.fr to /etc/apache2/sites-available/vh-stream-le-ssl.conf
Added an HTTP->HTTPS rewrite in addition to other RewriteRules; you may wish to check for overall consistency.
Congratulations! You have successfully enabled HTTPS on https://ludolpif.fr and https://www.ludolpif.fr

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
root@nuc1:~# vim /etc/apache2/sites-enabled/vh-stream.conf
root@nuc1:~# vim /etc/apache2/sites-enabled/vh-stream-le-ssl.conf
root@nuc1:~# rm /etc/apache2/sites-enabled/vh-stream-le-ssl.conf
root@nuc1:~# rm /etc/apache2/sites-available/vh-stream-le-ssl.conf
root@nuc1:~# a2ensite default-ssl
Enabling site default-ssl.
To activate the new configuration, you need to run:
  systemctl reload apache2
root@nuc1:~# systemctl reload apache2


ludolpif@lud-mn1:~$ alias curltrace='curl -vL --no-progress-meter -o /dev/null --stderr -'
ludolpif@lud-mn1:~$ curltrace ludolpif.fr/youtube | grep Location
< Location: https://ludolpif.fr/youtube
< Location: https://www.youtube.com/@ludolpif
ludolpif@lud-mn1:~$ curltrace ludolpif.fr/dISCord | grep Location
< Location: https://ludolpif.fr/dISCord
< Location: https://discord.gg/GfJ5RzcczV
ludolpif@lud-mn1:~$
```

## Configuration réseau pour homelab

```
# Configuration statique interface eno1 pour homelab (192.168.10.1/24, 2001:db8:0:a::1/64) + VLANs taggés
root@nuc1:~# editor /etc/network/interfaces
root@nuc1:~# \systemctl edit networking.service
[Service]
TimeoutStartSec=10s

root@nuc1:~# systemctl restart networking

root@nuc1:~# cat > /etc/sysctl.d/50-router.conf <<"EOT"
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
EOT
root@nuc1:~# systemctl restart systemd-sysctl


# Hack pour utiliser le systemd-resolved de nuc1 depuis les machines du homelab
# (et ne pas installer un cache DNS complet compatible DoT)
root@nuc1:~# editor /etc/systemd/resolved.conf
[Resolve]
DNS=80.67.169.40#ns1.fdn.fr 2001:910:800::40#ns.fdn.fr
FallbackDNS=80.67.169.12#ns0.fdn.fr 2001:910:800::12#ns0.fdn.fr
Domains=~.
DNSOverTLS=yes
LLMNR=no
DNSStubListenerExtra=192.168.1.42:53
DNSStubListenerExtra=192.168.10.1:53
DNSStubListenerExtra=[2001:db8:0:a::1]:53
ReadEtcHosts=no
root@nuc1:~# systemctl restart systemd-resolved

# Hack pour installer facilement des distro sur le homelab
# (internet sur le default vlan en dhcp)
root@nuc1:~# editor /etc/udhcpd.conf
root@nuc1:~# systemctl restart udhcpd

# Emettre des Router-Alerts IPv6 vers le homelab
root@nuc1:~# editor /etc/radvd.conf
root@nuc1:~# systemctl restart radvd

root@nuc1:~# editor /etc/wireguard/wg0.conf
# Voir my-vps.md (configuration simultanée des deux extrémités)

root@nuc1:~# editor /etc/nftables.conf
# Configuration avancée pour forward des réseaux homelab vers wireguard
root@nuc1:~# systemctl restart nftables
```

## Fix résolutions DNS lentes ou pas de réponse du tout si DoT est actif

Se produit dans le cas où on a dans `/etc/systemd/resolved.conf` :
```
[Resolve]
DNS=<un DNS externe>
FallbackDNS=<un DNS externe>
DNSOverTLS=yes
```
et une box qui annonce en DHCPv4/DHCPv6/RA un cache DNS qui n'écoute que sur le port 53 et pas 853 (DoT).

`resolved` a une notion de serveurs DNS globaux (définis dans `/etc/systemd/resolved.conf` et de serveur DNS par interface, définis par systemd-networkd ou d'autres (ifupdown, NetworkManager...). Ces derniers sont prioritaires par défaut. L'option `Domains=~.` peut aider à donner la priorité aux DNS de la section globale (si la valeur par interface de Domains est vide).

Si on utilise `systemd-networkd` et qu'on on veut absolument éviter des requêtes DNS vers des DNS appris via RA ou le DHCPv4 ou le DHCPv6, il faut forcer `UseDNS=false` dans 3 sections de `/etc/systemd/network/*.network`. Le `UseDNS=false` global dans [Network] ne marche pas pour moi (des DNS via le DHCPv6 sont appris quand même depuis ma box internet).

```
[DHCPv4]
UseDNS=false

[DHCPv6]
UseDNS=false

[IPv6AcceptRA]
UseDNS=false
```
