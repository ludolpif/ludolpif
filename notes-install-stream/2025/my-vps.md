## Installation debian 13

- Install debian 13 depuis Control Panel de l'hébergeur

```
$ ssh-copy-id ...
$ ssh root@my-vps # via la clé ssh

root@my-vps:~# editor /etc/ssh/sshd_config
# PasswordAuthentication no
root@my-vps:~# rm /etc/ssh/sshd_config.d/50-cloud-init.conf
root@my-vps:~# service ssh reload
```

## Fix bad Debian defaults pour OpenSSH et QoS (DSCP)

```
root@my-vps:~# echo 'IPQoS af21 cs0' > /etc/ssh/ssh_config.d/ssh_modern_IPQoS.conf
root@my-vps:~# echo 'IPQoS af21 cs0' > /etc/ssh/sshd_config.d/ssh_modern_IPQoS.conf
root@my-vps:~# service ssh reload
```

## Installation des tous les paquets utiles

L'utilitaire `dig` est dans le paquet `bind9-dnsutils`.
```
root@my-vps:~# apt install bind9-dnsutils conntrack iperf3 wireguard
```

## Personnalisation minimale pour moi et pour le stream

```
root@my-vps:~# update-alternatives --config editor
# J'aime choisir /usr/bin/vim.basic

root@my-vps:~# cat > /root/.vimrc <<"EOT"
runtime! defaults.vim
runtime! debian.vim
if &diff
	syntax off
else
	set mouse=
endif
set background=dark
EOT


root@my-vps:~# editor ~/.bashrc
# Décommenter pour avoir de la couleur et des alias
# Importer les sections pour .bash_aliases et bash_completion depuis /etc/skel/.bashrc

root@my-vps:~# editor ~/.bash_aliases
# Ajouter des fonctions *_stream() et des alias pour masquer
#  certaines valeurs dans les résultats de commandes interactives
```

## Suppression cloud-init

```
root@my-vps:~# apt autoremove --purge cloud-init cloud-initramfs-growroot

# Remarque, le réseau IPv4 et IPv6 est configuré avec netplan, ifupdown n'est pas installé

root@my-vps:~# cat /etc/netplan/50-cloud-init.yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        all-en:
            dhcp4: true
            dhcp6: true
            match:
                name: en*
    version: 2
root@my-vps:~#

```


## `dist-upgrade` en debian 13 avec nftables plutôt qu'iptables

```
root@my-vps:~# apt update
root@my-vps:~# apt install nftables iptables-
root@my-vps:~# apt upgrade
root@my-vps:~# editor /etc/apt/sources.list.d/debian.sources
# Remplacer bookworm par trixie, supprimer backports
root@my-vps:~# apt update
root@my-vps:~# apt dist-upgrade
root@my-vps:~# reboot
```

## Déploiement VPN Wireguard

```

# Il faut faire simultanément la config sur les deux machines, exemple ici : my-vps et nuc1

root@my-vps:~# mkdir /dev/shm/wireguard/
root@my-vps:~# chmod 0700 /dev/shm/wireguard/
root@my-vps:~# cd /dev/shm/wireguard/
root@my-vps:/dev/shm/wireguard# umask 077

root@my-vps:/dev/shm/wireguard# cat > gateway.example <<"EOT"
[Interface]
Address = 10.200.100.1/24
Address = fd10:200:100::1/64
ListenPort = 34567
PrivateKey = *******************************************=

[Peer]
PublicKey = *******************************************=
PresharedKey = *******************************************=
AllowedIPs = 10.200.100.42,fd10:200:100::42,192.168.10.0/24,2001:db8::/48
EOT
root@my-vps:/dev/shm/wireguard# cp gateway.example wg0.conf

root@nuc1:~# mkdir /dev/shm/wireguard/
root@nuc1:~# chmod 0700 /dev/shm/wireguard/
root@nuc1:~# cd /dev/shm/wireguard/
root@nuc1:/dev/shm/wireguard# umask 077

root@nuc1:/dev/shm/wireguard# cat > client.example <<"EOT"
[Interface]
Address = 10.200.100.42/24
Address = fd10:200:100::42/64
ListenPort = 34567
PrivateKey = *******************************************=

[Peer]
PublicKey = *******************************************=
PresharedKey = *******************************************=
AllowedIPs = 0.0.0.0/0,::/0
Endpoint = my-vps.example-domain.tld:34567
PersistentKeepalive = 25
EOT
root@nuc1:/dev/shm/wireguard# cp client.example wg0.conf

# https://www.wireguard.com/quickstart/#key-generation
# Chaque extrémité du tunnel utilise sa propre paire de clé publique+privée
# On peut ajouter une pre-shared key, commune aux deux extrémités

root@my-vps:/dev/shm/wireguard# wg genkey > privatekey
root@my-vps:/dev/shm/wireguard# cat privatekey >> wg0.conf
root@my-vps:/dev/shm/wireguard# nano wg0.conf
# Déplacer la clé privée de la dernière ligne du fichier au bon endroit

root@nuc1:/dev/shm/wireguard# wg genkey > privatekey
root@nuc1:/dev/shm/wireguard# cat privatekey >> wg0.conf
root@nuc1:/dev/shm/wireguard# nano wg0.conf
# Déplacer la clé privée de la dernière ligne du fichier au bon endroit

root@my-vps:/dev/shm/wireguard# wg pubkey < privatekey
# Renseigner la publickey générée depuis my-vps dans wg0.conf à l'autre bout du tunnel (sur nuc1)
root@nuc1:/dev/shm/wireguard# nano wg0.conf

root@nuc1:/dev/shm/wireguard# wg pubkey < privatekey
# Renseigner la publickey générée depuis nuc1 dans wg0.conf à l'autre bout du tunnel (sur my-vps)
root@my-vps:/dev/shm/wireguard# nano wg0.conf

# A générer une seule fois (par exemple sur my-vps), et à copier dans les deux fichiers wg0.conf
root@my-vps:/dev/shm/wireguard# wg genpsk >> wg0.conf
root@my-vps:/dev/shm/wireguard# nano wg0.conf
# Déplacer la clé pré-partagée de la dernière ligne du fichier au bon endroit.
# Renseigner également la pré-partagée à l'autre bout du tunnel dans ce fichier (sur nuc1)
root@nuc1:/dev/shm/wireguard# nano wg0.conf

root@my-vps:/dev/shm/wireguard# rm privatekey publickey
root@my-vps:/dev/shm/wireguard# mv wg0.example wg0.conf /etc/wireguard/
root@my-vps:/dev/shm/wireguard# cd
root@my-vps:~# systemctl enable wg-quick@wg0
root@my-vps:~# systemctl start wg-quick@wg0
root@my-vps:~# systemctl status wg-quick@wg0

root@nuc1:/dev/shm/wireguard# rm privatekey publickey
root@nuc1:/dev/shm/wireguard# mv wg0.example wg0.conf /etc/wireguard/
root@nuc1:/dev/shm/wireguard# cd
root@nuc1:~# systemctl enable wg-quick@wg0
root@nuc1:~# systemctl start wg-quick@wg0
root@nuc1:~# systemctl status wg-quick@wg0

root@my-vps:~# wg
interface: wg0
  public key: (hidden)
  private key: (hidden)
  listening port: 34567

peer: (hidden)
  preshared key: (hidden)
  endpoint: (hidden)
  allowed ips: 10.200.100.42/32, fd10:200:100::42/128
  latest handshake: 8 seconds ago
  transfer: 4.3 KiB received, 3.2 KiB sent

root@my-vps:~# ping -c2 fd10:200:100::42
PING fd10:200:100::42 (fd10:200:100::42) 56 data bytes
64 bytes from fd10:200:100::42: icmp_seq=1 ttl=64 time=30.9 ms
64 bytes from fd10:200:100::42: icmp_seq=2 ttl=64 time=30.9 ms

--- fd10:200:100::42 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 30.883/30.908/30.933/0.025 ms
root@my-vps:~#

root@nuc1:~# wg
interface: wg0
  public key: (hidden)
  private key: (hidden)
  listening port: 34567
  fwmark: 0xca6c

peer: (hidden)
  preshared key: (hidden)
  endpoint: (hidden)
  allowed ips: 0.0.0.0/0, ::/0
  latest handshake: 14 seconds ago
  transfer: 3.2 KiB received, 4.3 KiB sent
  persistent keepalive: every 25 seconds

root@nuc1:~# ping -c2 10.200.100.1
PING 10.200.100.1 (10.200.100.1) 56(84) bytes of data.
64 bytes from 10.200.100.1: icmp_seq=1 ttl=64 time=30.9 ms
64 bytes from 10.200.100.1: icmp_seq=2 ttl=64 time=30.5 ms

--- 10.200.100.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 30.527/30.694/30.861/0.167 ms
root@nuc1:~#


root@my-vps:~# cat >/etc/nftables.conf <<"EOT"
#!/usr/sbin/nft -f

define if_internet = ens6

flush ruleset

# https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)
table inet nat {
        chain postrouting {
                type nat hook postrouting priority srcnat;
                iifname wg0 oifname $if_internet counter masquerade comment "from-home-to-internet source NAT"
        }
}

table inet filter {
        chain input {
                type filter hook input priority filter; policy accept;
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                # Accept already estalished connexions (and replies and related flows)
                ct state vmap { established : accept, related : accept, invalid : drop }
                # Accept from-home-to-internet
                iifname wg0 oifname $if_internet ct state new counter accept
        }
        chain output {
                type filter hook output priority filter; policy accept;
        }
}
EOT

root@my-vps:~# systemctl enable nftables
root@my-vps:~# systemctl restart nftables

root@my-vps:~# cat > /etc/sysctl.d/50-router.conf <<"EOT"
net.ipv4.conf.all.forwarding = 1
net.ipv6.conf.all.forwarding = 1
EOT
root@my-vps:~# systemctl restart systemd-sysctl
```

- Tester depuis nuc1 que le forwarding + masquerade fonctionne (pour n'importe quelle destination vers internet)

```
root@nuc1:~# ping -c2 2001:4860:4860::8888
PING 2001:4860:4860::8888(2001:4860:4860::8888) 56 data bytes
64 bytes from 2001:4860:4860::8888: icmp_seq=1 ttl=114 time=42.2 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=2 ttl=114 time=40.7 ms

--- 2001:4860:4860::8888 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 40.738/41.465/42.192/0.727 ms

root@nuc1:~# ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=114 time=41.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=114 time=43.7 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 41.175/42.433/43.691/1.258 ms
root@nuc1:~#
```

- TODO : recurseur DNS supportant DoT

## Bonus Règles nftables avancées pour auto-hébergement

`/etc/nftables.conf:`

```
#!/usr/sbin/nft -f

define if_internet = ens6

flush ruleset

# https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)
table inet nat {
        chain prerouting {
                type nat hook prerouting priority dstnat;
                #iifname $if_internet tcp dport {80,443} dnat ip to 192.168.10.1 comment "from-internet-to-home example"
                #iifname $if_internet tcp dport {80,443} dnat ip6 to 2001:db8:0:a::1 comment "from-internet-to-home example"
        }
        chain postrouting {
                type nat hook postrouting priority srcnat;
                iifname wg0 oifname $if_internet counter masquerade comment "from-home-to-internet source NAT"
        }
}

table inet filter {
        chain input {
                type filter hook input priority filter; policy accept;
                # http://shouldiblockicmp.com/#RateLim
                ip protocol icmp limit rate 50/second counter accept
                ip protocol icmp counter drop
                ip6 nexthdr icmpv6 limit rate 50/second counter accept
                ip6 nexthdr icmpv6 counter drop
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                # http://shouldiblockicmp.com/#RateLim
                ip protocol icmp limit rate 50/second counter accept
                ip protocol icmp counter drop
                ip6 nexthdr icmpv6 limit rate 50/second counter accept
                ip6 nexthdr icmpv6 counter drop
                # Accept already estalished connexions
                ct state vmap { established : accept, related : accept, invalid : drop }
                iifname wg0 oifname $if_internet ct state new jump forward-from-wg0-to-internet
                iifname $if_internet oifname wg0 ct state new jump forward-from-internet-to-home
        }
        chain forward-from-wg0-to-internet {
                counter accept
        }
        chain forward-from-internet-to-home {
                #ip daddr 192.168.10.1 tcp dport {80,443} counter accept comment "from-internet-to-home example"
                #ip6 daddr 2001:db8:0:a::1 tcp dport {80,443} counter accept comment "from-internet-to-home example"
                #counter log prefix "forward-from-internet-to-home drop: " drop
        }
        chain output {
                type filter hook output priority filter; policy accept;
        }
}
EOT
```
