# 🔥 Configuration pfSense

## Sommaire
- [Présentation](#présentation)
- [Installation](#installation)
- [Configuration réseau](#configuration-réseau)
- [Règles Firewall](#règles-firewall)
- [NAT et redirection de ports](#nat-et-redirection-de-ports)
- [WireGuard sur pfSense](#wireguard-sur-pfsense)
- [Bonnes pratiques appliquées](#bonnes-pratiques-appliquées)
- [Commandes utiles](#commandes-utiles)

---

## Présentation

pfSense est une distribution open source basée sur FreeBSD, spécialisée dans les fonctions de firewall et de routeur. C'est une solution professionnelle utilisée aussi bien dans les PME que dans les grandes entreprises.

Dans ce lab, pfSense tourne sur un **Lenovo ThinkPad T430** upgradé (16 Go RAM, SSD 240 Go Kingston) et joue plusieurs rôles simultanément :

| Rôle | Description |
|------|-------------|
| 🔥 Firewall | Filtre tout le trafic entrant et sortant |
| 🔀 Routeur | Route les paquets entre WAN, LAN et VPN |
| 🔁 NAT | Traduit les adresses entre réseau public et privé |
| 🔐 Serveur VPN | Héberge le serveur WireGuard |
| 🛡️ Point de contrôle | Tout le trafic du lab passe obligatoirement par lui |

---

## Installation

### Prérequis matériels
- Lenovo ThinkPad T430 avec SSD Kingston 240 Go installé
- Adaptateur USB-Ethernet pour la seconde interface réseau
- Clé USB bootable pfSense CE

### Création de la clé USB bootable
1. Télécharger l'image pfSense CE sur [pfsense.org](https://www.pfsense.org/download/)
2. Flasher l'image sur une clé USB avec **Balena Etcher** ou **Rufus**
3. Démarrer le T430 sur la clé USB (modifier l'ordre de boot dans le BIOS)

### Processus d'installation
1. Démarrer sur la clé USB → sélectionner **Install pfSense**
2. Choisir le SSD Kingston comme disque de destination
3. Laisser l'installation se terminer et retirer la clé USB
4. Au premier démarrage, pfSense lance l'assistant de configuration console

### Attribution des interfaces
Au démarrage, pfSense demande d'assigner les interfaces :

```
WAN → em0 (interface réseau interne du T430) → branchée sur la box FAI
LAN → ue0 (adaptateur USB-Ethernet)          → branchée sur le switch
```

> 💡 C'est l'inverse de ce qu'on pourrait intuitivement penser : l'interface **interne** du T430 sert de WAN, et l'adaptateur **USB-Ethernet** sert de LAN vers le switch.

---

## Configuration réseau

### Interface WAN

| Paramètre | Valeur |
|-----------|--------|
| Interface | em0 (carte réseau interne) |
| Type | DHCP (adresse attribuée par la box FAI) |
| Réseau | 192.168.1.0/24 |
| IP pfSense | 192.168.1.x |
| Passerelle | 192.168.1.1 (box FAI) |

### Interface LAN

| Paramètre | Valeur |
|-----------|--------|
| Interface | ue0 (adaptateur USB-Ethernet) |
| Type | Static |
| IP pfSense | 192.168.10.x |
| Masque | /24 |
| DHCP Server | Activé — distribue les IPs aux machines du lab |

### Interface WireGuard (tun_wg0)

| Paramètre | Valeur |
|-----------|--------|
| Interface | tun_wg0 |
| IP pfSense | 10.0.0.x |
| Masque | /24 |
| Rôle | Tunnel VPN pour accès distant |

---

## Règles Firewall

pfSense applique une **politique de refus par défaut** : tout ce qui n'est pas explicitement autorisé est bloqué. Les règles sont évaluées de haut en bas, la première règle qui correspond s'applique.

### Interface WAN

| Action | Protocole | Source | Destination | Port | Description |
|--------|-----------|--------|-------------|------|-------------|
| Pass | UDP | Any | WAN address | 51820 | WireGuard entrant |
| Block | Any | Any | Any | Any | Tout bloquer (défaut) |

> ⚠️ Seul le port WireGuard est ouvert sur le WAN. Proxmox et les VMs ne sont **jamais** directement exposés sur Internet.

### Interface LAN

| Action | Protocole | Source | Destination | Port | Description |
|--------|-----------|--------|-------------|------|-------------|
| Pass | Any | LAN net | Any | Any | LAN accès complet |

> 💡 Les machines du LAN (Proxmox, Ryzen 5) ont accès à tout. Le contrôle se fait principalement sur le WAN et WireGuard.

### Interface WireGuard

| Action | Protocole | Source | Destination | Description |
|--------|-----------|--------|-------------|-------------|
| Pass | Any | WireGuard net | LAN net | VPN → accès au lab |
| Pass | Any | WireGuard net | WireGuard net | Communication entre peers |
| Block | Any | Any | Any | Tout bloquer (défaut) |

> 💡 Un client VPN connecté peut accéder au lab (`192.168.10.x`) mais ne peut pas sortir sur Internet via pfSense. C'est volontaire : le split tunneling est géré côté client.

---

## NAT et redirection de ports

### NAT Outbound (sortant)
pfSense effectue du **NAT automatique** pour toutes les machines du LAN : leurs IPs privées (`192.168.10.x`) sont traduites en IP WAN avant de sortir vers Internet.

```
Machine LAN (192.168.10.x) → pfSense NAT → IP WAN (192.168.1.x) → Internet
```

### Port Forward (entrant)
Un seul port est redirigé depuis la box FAI vers pfSense :

| Protocole | Port externe | Destination | Port interne | Service |
|-----------|-------------|-------------|--------------|---------|
| UDP | 51820 | 192.168.1.x (pfSense WAN) | 51820 | WireGuard |

> ⚠️ Cette redirection est configurée **sur la box FAI**, pas dans pfSense. pfSense reçoit le trafic WireGuard directement sur son interface WAN.

---

## WireGuard sur pfSense

Voir la documentation complète : [04-wireguard.md](04-wireguard.md)

### Installation du package
**System → Package Manager → Available Packages** → chercher `WireGuard` → Install

### Résumé de la configuration
- **Tunnel** : `tun_wg0` — VPN-LAB — port UDP 51820
- **IP tunnel** : `10.0.0.x/24`
- **Peers** :
  - iPhone (`10.0.0.x`) — Dynamic Endpoint — Keepalive 25s
  - Laptop HP Linux Mint (`10.0.0.x`) — Dynamic Endpoint — Keepalive 25s

### Vérifier l'état du tunnel
**VPN → WireGuard → Status**

Un tunnel actif affiche :
```
tun_wg0    listening port: 51820    peers: 2
peer: <clé_publique>    latest handshake: X seconds ago
```

---

## Bonnes pratiques appliquées

### Principe du moindre privilège
- Seul le port 51820 UDP est ouvert sur le WAN
- Proxmox n'est **jamais** accessible directement depuis Internet
- Les VMs ne sont accessibles que depuis le LAN ou via VPN

### Isolation réseau
- Le réseau lab `192.168.10.x` est totalement séparé du réseau domestique `192.168.1.x`
- pfSense est le **seul point de passage** entre les deux réseaux
- Sans règle firewall explicite, rien ne passe

### Mises à jour
Maintenir pfSense à jour est essentiel :
**System → Update → Check for Updates**

---

## Commandes utiles

### Depuis la console pfSense (accès physique ou SSH)

```bash
# Voir les interfaces réseau
ifconfig

# Voir la table de routage
netstat -rn

# Ping depuis pfSense
ping 192.168.10.x

# Voir les connexions WireGuard actives
wg show
```

### Depuis l'interface web

| Action | Chemin |
|--------|--------|
| Voir les logs firewall | Status → System Logs → Firewall |
| Voir le trafic en temps réel | Diagnostics → Traffic Graph |
| Tester la connectivité | Diagnostics → Ping |
| Voir les baux DHCP | Status → DHCP Leases |
| État des interfaces | Status → Interfaces |
| État WireGuard | VPN → WireGuard → Status |
