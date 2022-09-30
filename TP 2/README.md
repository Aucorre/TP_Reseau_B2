# TP2 : Ethernet, IP, et ARP

Dans ce TP on va approfondir trois protocoles, qu'on a survolé jusqu'alors :

- **IPv4** _(Internet Protocol Version 4)_ : gestion des adresses IP
  - on va aussi parler d'ICMP, de DHCP, bref de tous les potes d'IP quoi !
- **Ethernet** : gestion des adresses MAC
- **ARP** _(Address Resolution Protocol)_ : permet de trouver l'adresse MAC de quelqu'un sur notre réseau dont on connaît l'adresse IP

# Sommaire

- [TP2 : Ethernet, IP, et ARP](#tp2--ethernet-ip-et-arp)
- [Sommaire](#sommaire)
- [I. Setup IP](#i-setup-ip)
- [II. ARP my bro](#ii-arp-my-bro)
- [III. DHCP you too my brooo](#iii-dhcp-you-too-my-brooo)
- [IV. Avant-goût TCP et UDP](#iv-avant-goût-tcp-et-udp)

# I. Setup IP

```
netsh interface ipv4 set address name="Wi-Fi" static 10.33.18.69 255.255.255.192 10.33.19.254

nano /etc/sysconfig/network-scripts/ifcfg-enp0s8


ipconfig
Carte réseau sans fil Wi-Fi :

   Suffixe DNS propre à la connexion. . . :
   Adresse IPv6 de liaison locale. . . . .: fe80::5956:481c:d0f0:8c03%14
   Adresse IPv4. . . . . . . . . . . . . .: 10.33.18.69
   Masque de sous-réseau. . . . . . . . . : 255.255.255.192
   Passerelle par défaut. . . . . . . . . : 10.33.19.254
```

L'adresse de broadcast est 10.33.18.255

L'adresse de réseau est 10.33.18.0

```
ip a
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:77:3c:bd brd ff:ff:ff:ff:ff:ff
    inet 10.250.1.2/24 brd 10.250.1.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe77:3cbd/64 scope link
       valid_lft forever preferred_lft forever
```

L'adresse de broadcast est 10.250.1.255

L'adresse de réseau est 10.250.1.0

🌞 **Prouvez que la connexion est fonctionnelle entre les deux machines**

```
ping 10.250.1.2

Envoi d’une requête 'Ping'  10.250.1.2 avec 32 octets de données :
Réponse de 10.250.1.2 : octets=32 temps<1ms TTL=64
Réponse de 10.250.1.2 : octets=32 temps<1ms TTL=64

Statistiques Ping pour 10.250.1.2:
    Paquets : envoyés = 2, reçus = 2, perdus = 0 (perte 0%),
Durée approximative des boucles en millisecondes :
    Minimum = 0ms, Maximum = 0ms, Moyenne = 0ms
```

🌞 **Wireshark it**

🦈 **PCAP qui contient les paquets ICMP qui vous ont permis d'identifier les types ICMP**

[ping](pings.pcapng)

# II. ARP my bro

## La suite du tp a été réalisé avec une autre adresse ip et passerelle pour la carte Wi-Fi de la machine 1

🌞 **Check the ARP table**

```
 arp -a

Interface?: 10.250.1.1 --- 0x2
  Adresse Internet      Adresse physique      Type
  10.250.1.2            08-00-27-77-3c-bd     dynamique
  10.250.1.255          ff-ff-ff-ff-ff-ff     statique
  224.0.0.2             01-00-5e-00-00-02     statique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
  255.255.255.255       ff-ff-ff-ff-ff-ff     statique

Interface?: 192.168.1.41 --- 0xe
  Adresse Internet      Adresse physique      Type
  192.168.1.47          a8-d3-f7-f4-b2-2a     dynamique
  192.168.1.254         7c-26-64-d6-58-1c     dynamique
  192.168.1.255         ff-ff-ff-ff-ff-ff     statique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique
  224.0.0.252           01-00-5e-00-00-fc     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
  255.255.255.255       ff-ff-ff-ff-ff-ff     statique
```

La mac de la machine 2 est 08-00-27-77-3c-bd

La mac de la gateway est 7c-26-64-d6-58-1c

🌞 **Manipuler la table ARP**

```
arp -d
arp -a

Interface?: 10.250.1.1 --- 0x2
  Adresse Internet      Adresse physique      Type
  224.0.0.22            01-00-5e-00-00-16     statique

Interface?: 192.168.1.41 --- 0xe
  Adresse Internet      Adresse physique      Type
  192.168.1.47          a8-d3-f7-f4-b2-2a     dynamique
  192.168.1.254         7c-26-64-d6-58-1c     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
- ré-effectuez des pings, et constatez la ré-apparition des données dans la table ARP
```

```
ping 10.250.1.2

Envoi d’une requête 'Ping'  10.250.1.2 avec 32 octets de données :
Réponse de 10.250.1.2 : octets=32 temps<1ms TTL=64

[...]
arp -a

Interface?: 10.250.1.1 --- 0x2
  Adresse Internet      Adresse physique      Type
  10.250.1.2            08-00-27-77-3c-bd     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  224.0.0.251           01-00-5e-00-00-fb     statique

Interface?: 192.168.1.41 --- 0xe
  Adresse Internet      Adresse physique      Type
  192.168.1.47          a8-d3-f7-f4-b2-2a     dynamique
  192.168.1.254         7c-26-64-d6-58-1c     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
```

🌞 **Wireshark it**

La première trame est de type ARP Broadcast, elle est envoyée par la machine 1 vers tout le monde pour connaitre l'adresse mac de la machine 2.

La deuxième trame est de type ARP Reply, elle est envoyée par la machine 2 vers la machine 1 pour lui donner son adresse mac.

🦈 **PCAP qui contient les trames ARP**

[echanges arp](arp.pcapng)

# III. DHCP you too my brooo

Quand on arrive dans un réseau, notre PC contacte un serveur DHCP, et récupère généralement 3 infos :

- **1.** une IP à utiliser
- **2.** l'adresse IP de la passerelle du réseau
- **3.** l'adresse d'un serveur DNS joignable depuis ce réseau

🌞 **Wireshark it**

La trame 1 est de type DHCP Discover, elle est envoyée par la machine encore sans ip vers tout le monde pour connaitre l'adresse ip du serveur DHCP.

On voit le 0.0.0.0 comme source et 255.255.255.255 comme destination

La trame 2 est de type DHCP Offer, elle est envoyée par le serveur DHCP vers la machine pour lui proposer une adresse ip. (On a ici l'adresse de la passerelle du réseau et l'adresse d'un serveur DNS joignable depuis ce réseau)

La trame 3 est de type DHCP Request, elle est envoyée par la machine vers le serveur DHCP pour lui demander l'adresse ip proposée.(On a ici l'adresse ip d'utilsateur)

La trame 4 est de type DHCP ACK, elle est envoyée par le serveur DHCP vers la machine pour lui confirmer l'adresse ip proposée.

🦈 **PCAP qui contient l'échange DORA**

[dhcp](dhcp.pcapng)

# IV. Avant-goût TCP et UDP

---

🌞 **Wireshark it**

Notre PC se connecte à l'adresse 213.44.16.77 sur le port 443.

🦈 **PCAP qui contient un extrait de l'échange qui vous a permis d'identifier les infos**

[échange](youtube.pcapng)
