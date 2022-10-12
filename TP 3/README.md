# TP3 : On va router des trucs

Au menu de ce TP, on va revoir un peu ARP et IP histoire de **se mettre en jambes dans un environnement avec des VMs**.

Puis on mettra en place **un routage simple, pour permettre à deux LANs de communiquer**.

## Sommaire

- [TP3 : On va router des trucs](#tp3--on-va-router-des-trucs)
  - [Sommaire](#sommaire)
  - [I. ARP](#i-arp)
    - [1. Echange ARP](#1-echange-arp)
    - [2. Analyse de trames](#2-analyse-de-trames)
  - [II. Routage](#ii-routage)
    - [1. Mise en place du routage](#1-mise-en-place-du-routage)
    - [2. Analyse de trames](#2-analyse-de-trames-1)
    - [3. Accès internet](#3-accès-internet)
  - [III. DHCP](#iii-dhcp)
    - [1. Mise en place du serveur DHCP](#1-mise-en-place-du-serveur-dhcp)
    - [2. Analyse de trames](#2-analyse-de-trames-2)

## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
| -------- | ------------- |
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

### 1. Echange ARP

🌞**Générer des requêtes ARP**

```
ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=0.435 ms

ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=0.345 ms
```

```
ip neigh show
10.3.1.12 dev enp0s8 lladdr 08:00:27:50:0e:86 STALE
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:02 DELAY

ip neigh show
10.3.1.11 dev enp0s8 lladdr 08:00:27:1d:09:74 STALE
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:02 DELAY
```

```
ip neigh show
10.3.1.12 dev enp0s8 lladdr 08:00:27:50:0e:86 STALE

ip a
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:50:0e:86 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.12/24 brd 10.3.1.255 scope global noprefixroute enp0s8
[...]
```

L'adresse mac de marcel est `08:00:27:50:0e:86` et celle de john est `08:00:27:1d:09:74`.

### 2. Analyse de trames

🌞**Analyse de trames**

```
sudo ip neigh flush all

sudo tcpdump -c 4 -i enp0s8 -w ping.pcapng arp
dropped privs to tcpdump
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
4 packets captured
5 packets received by filter
0 packets dropped by kernel
```

```
sudo ip neigh flush all

ping 1 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=0.538 ms

--- 10.3.1.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.538/0.538/0.538/0.000 ms
```

🦈 trames [ARP](ping.pcap)

## II. Routage

```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ca:30:d7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86262sec preferred_lft 86262sec
    inet6 fe80::a00:27ff:feca:30d7/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9d:93:e0 brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.254/24 brd 10.3.1.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9d:93e0/64 scope link
       valid_lft forever preferred_lft forever
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:85:2b:df brd ff:ff:ff:ff:ff:ff
    inet 10.3.2.254/24 brd 10.3.2.255 scope global noprefixroute enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe85:2bdf/64 scope link
       valid_lft forever preferred_lft forever
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

```
[audy@routeur ~]sudo firewall-cmd --get-active-zone
public
  interfaces: enp0s3 enp0s8 enp0s9

[audy@routeur ~]sudo firewall-cmd --add-masquerade --zone=public --permanent
success
```

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**

```
[audy@john ~]$ sudo ip route add 10.3.2.0/24 via 10.3.1.254 dev enp0s8
[audy@marcel ~]$ sudo ip route add 10.3.1.0/24 via 10.3.2.254 dev enp0s8
```

```
[audy@john ~]$ ping -c 1 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=1.63 ms

--- 10.3.2.12 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.628/1.628/1.628/0.000 ms
```

### 2. Analyse de trames

🌞**Analyse des échanges ARP**

```
[audy@john ~]$ sudo ip neigh flush all
[audy@john ~]$ ping -c 1 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=1.45 ms
[...]
[audy@john ~]$ ip neigh show
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:02 REACHABLE
10.3.1.254 dev enp0s8 lladdr 08:00:27:9d:93:e0 REACHABLE

[audy@routeur ~]$ sudo ip neigh flush all
[audy@routeur ~]$ ip neigh show
10.3.1.11 dev enp0s8 lladdr 08:00:27:1d:09:74 REACHABLE
10.3.2.12 dev enp0s9 lladdr 08:00:27:50:0e:86 REACHABLE
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:02 REACHABLE

[audy@marcel ~]$ sudo ip neigh flush all
[audy@marcel ~]$ ip neigh show
10.3.2.1 dev enp0s8 lladdr 0a:00:27:00:00:16 REACHABLE
10.3.2.254 dev enp0s8 lladdr 08:00:27:85:2b:df REACHABLE
```

```
[audy@marcel ~]$ sudo tcpdump -c 4 -i enp0s8 -w arp.pcapng arp
dropped privs to tcpdump
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

| ordre | type trame  | IP source  | MAC source                    | IP destination | MAC destination               |
| ----- | ----------- | ---------- | ----------------------------- | -------------- | ----------------------------- |
| 1     | Requête ARP | x          | `john` `08:00:27:1d:09:74`    | x              | Broadcast `FF:FF:FF:FF:FF`    |
| 2     | Réponse ARP | x          | `marcel` ``08:00:27:50:0e:86` | x              | `routeur` `08:00:27:85:2b:df` |
| 3     | Ping        | 10.3.2.254 | `routeur` `08:00:27:85:2b:df` | 10.3.2.12      | `marcel` ``08:00:27:50:0e:86` |
| 4     | Pong        | 10.3.2.12  | `marcel` `08:00:27:85:2b:df`  | 10.3.2.254     | `routeur` `08:00:27:50:0e:86` |
| 5     | Requête ARP | x          | `marcel` `08:00:27:85:2b:df`  | x              | Broadcast `FF:FF:FF:FF:FF`    |
| 6     | Réponse ARP | x          | `john` `08:00:27:1d:09:74`    | x              | `routeur` `08:00:27:85:2b:df` |

🦈 trames [ARP](arp.pcap)

### 3. Accès internet

🌞**Donnez un accès internet à vos machines**

```
[audy@john ~]$ sudo ip route add default via 10.3.1.254 dev enp0s8
[audy@john ~]$ ping -c 1 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=53 time=21.5 ms

--- 1.1.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 21.497/21.497/21.497/0.000 ms
```

```
[audy@marcel ~]$ ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=22.0 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 22.008/22.008/22.008/0.000 ms
```

```
[audy@john ~]$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1

[audy@john ~]$ dig google.com

; <<>> DiG 9.16.23-RH <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 65062
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             223     IN      A       142.250.178.142

;; Query time: 30 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed Oct 12 11:13:01 CEST 2022
;; MSG SIZE  rcvd: 55

[audy@john ~]$ ping ynov.com
PING ynov.com (104.26.10.233) 56(84) bytes of data.
64 bytes from 104.26.10.233 (104.26.10.233): icmp_seq=1 ttl=53 time=21.4 ms
[...]
```

🌞**Analyse de trames**

```
[audy@john ~]$ sudo tcpdump -i enp0s8 -c 2 -w tp2_routage_internet.pcapng icmp &
[1] 10999
[audy@john ~]$ dropped privs to tcpdump
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=22.8 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 22.821/22.821/22.821/0.000 ms
[audy@john ~]$ 2 packets captured
3 packets received by filter
0 packets dropped by kernel
```

| ordre | type trame | IP source          | MAC source                    | IP destination     | MAC destination               |     |
| ----- | ---------- | ------------------ | ----------------------------- | ------------------ | ----------------------------- | --- |
| 1     | ping       | `john` `10.3.1.11` | `john` `08:00:27:1d:09:74`    | `8.8.8.8`          | `routeur` `08:00:27:85:2b:df` |     |
| 2     | pong       | `8.8.8.8`          | `routeur` `08:00:27:85:2b:df` | `john` `10.3.1.11` | `john` `08:00:27:1d:09:74`    | ... |

🦈 **Capture réseau `tp2_routage_internet.pcapng`**

## III. DHCP

On reprend la config précédente, et on ajoutera à la fin de cette partie une 4ème machine pour effectuer des tests.

| Machine  | `10.3.1.0/24`              | `10.3.2.0/24` |
| -------- | -------------------------- | ------------- |
| `router` | `10.3.1.254`               | `10.3.2.254`  |
| `john`   | `10.3.1.11`                | no            |
| `bob`    | oui mais pas d'IP statique | no            |
| `marcel` | no                         | `10.3.2.12`   |

```schema
   john               router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └─┬─┘    └─────┘    └───┘    └─────┘
   john        │
  ┌─────┐      │
  │     │      │
  │     ├──────┘
  └─────┘
```

### 1. Mise en place du serveur DHCP

🌞**Sur la machine `john`, vous installerez et configurerez un serveur DHCP** (go Google "rocky linux dhcp server").

```
[audy@john ~]$ sudo cat /etc/dhcp/dhcpd.conf
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp-server/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#

default-lease-time 900;
max-lease-time 10800;
ddns-update-style none;
authoritative;
subnet 10.3.1.0 netmask 255.255.255.0 {
range 10.3.1.1 10.3.1.253;
option routers 10.3.1.254;
option subnet-mask 255.255.255.0;
option domain-name-servers 8.8.8.8;

[audy@john ~]$ sudo systemctl enable dhcpd
Created symlink /etc/systemd/system/multi-user.target.wants/dhcpd.service → /usr/lib/systemd/system/dhcpd.service.
```

> Il est possible d'utilise la commande `dhclient` pour forcer à la main, depuis la ligne de commande, la demande d'une IP en DHCP, ou renouveler complètement l'échange DHCP (voir `dhclient -h` puis call me et/ou Google si besoin d'aide).

🌞**Améliorer la configuration du DHCP**

- ajoutez de la configuration à votre DHCP pour qu'il donne aux clients, en plus de leur IP :
  - une route par défaut
  - un serveur DNS à utiliser
- récupérez de nouveau une IP en DHCP sur `marcel` pour tester :
  - `marcel` doit avoir une IP
    - vérifier avec une commande qu'il a récupéré son IP
    - vérifier qu'il peut `ping` sa passerelle
  - il doit avoir une route par défaut
    - vérifier la présence de la route avec une commande
    - vérifier que la route fonctionne avec un `ping` vers une IP
  - il doit connaître l'adresse d'un serveur DNS pour avoir de la résolution de noms
    - vérifier avec la commande `dig` que ça fonctionne
    - vérifier un `ping` vers un nom de domaine

### 2. Analyse de trames

🌞**Analyse de trames**

- lancer une capture à l'aide de `tcpdump` afin de capturer un échange DHCP
- demander une nouvelle IP afin de générer un échange DHCP
- exportez le fichier `.pcapng`

🦈 **Capture réseau `tp2_dhcp.pcapng`**
