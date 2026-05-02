# 🖥️ Configuration Proxmox VE

## Sommaire
- [Présentation](#présentation)
- [Installation](#installation)
- [Configuration réseau](#configuration-réseau)
- [Machines virtuelles](#machines-virtuelles)
- [VM 100 — Kali Linux](#vm-100--kali-linux)
- [VM 101 — Windows Server](#vm-101--windows-server)
- [VM 102 — Windows Client](#vm-102--windows-client)
- [Gestion des snapshots](#gestion-des-snapshots)
- [Bonnes pratiques appliquées](#bonnes-pratiques-appliquées)
- [Commandes utiles](#commandes-utiles)

---

## Présentation

Proxmox VE (Virtual Environment) est un hyperviseur bare-metal open source basé sur Debian. Il permet de créer et gérer des machines virtuelles (KVM) et des conteneurs (LXC) depuis une interface web.

Dans ce lab, Proxmox tourne sur le **PC Ryzen 7** (32 Go RAM, SSD 1 To) et héberge l'ensemble des machines virtuelles utilisées pour les exercices offensifs et défensifs.

| Caractéristique | Détail |
|----------------|--------|
| Version | Proxmox VE 9.1 |
| Hyperviseur | KVM (Kernel-based Virtual Machine) |
| Interface | Web — `https://192.168.10.x:8006` |
| Accès | Via VPN WireGuard ou LAN direct |

---

## Installation

### Prérequis
- PC Ryzen 7 avec SSD 1 To
- Clé USB bootable Proxmox VE
- Connexion au switch LAN (derrière pfSense)

### Création de la clé USB bootable
1. Télécharger l'ISO Proxmox VE sur [proxmox.com](https://www.proxmox.com/en/downloads)
2. Flasher l'image avec **Balena Etcher** ou **Rufus**
3. Démarrer le PC sur la clé USB (modifier l'ordre de boot dans le BIOS/UEFI)

### Processus d'installation
1. Sélectionner **Install Proxmox VE**
2. Choisir le SSD 1 To comme disque de destination
3. Configurer :
   - **IP** : adresse statique dans le réseau `192.168.10.x/24`
   - **Passerelle** : `192.168.10.x` (pfSense LAN)
   - **DNS** : `192.168.10.x` (pfSense fait aussi office de DNS)
4. Finaliser l'installation et redémarrer
5. Accéder à l'interface web : `https://192.168.10.x:8006`

> 💡 Le certificat SSL de Proxmox est auto-signé — le navigateur affiche un avertissement "Non sécurisé", c'est normal. Accepter l'exception pour continuer.

---

## Configuration réseau

### Interface réseau Proxmox

Proxmox utilise une **bridge Linux** (`vmbr0`) qui connecte les VMs au réseau physique :

```
Carte réseau physique (eth0)
        │
   ┌────▼────┐
   │  vmbr0  │  ← bridge Linux Proxmox
   └────┬────┘
        │
   ┌────┴────┬──────────┐
   │         │          │
 VM 100    VM 101     VM 102
 Kali      WinSrv    WinClt
```

Toutes les VMs connectées à `vmbr0` sont dans le réseau `192.168.10.x/24` et passent par pfSense pour accéder à Internet ou être jointes depuis l'extérieur.

### Configuration vmbr0

| Paramètre | Valeur |
|-----------|--------|
| Bridge | vmbr0 |
| IP Proxmox | 192.168.10.x/24 |
| Passerelle | 192.168.10.x (pfSense) |
| Ports | Interface physique Ethernet |

---

## Machines virtuelles

### Vue d'ensemble

| VM ID | Nom | OS | Rôle | RAM | Disque |
|-------|-----|----|------|-----|--------|
| 100 | kali | Kali Linux | Pentest / Attaque | 4 Go | 40 Go |
| 101 | win-server | Windows Server | Active Directory | 4 Go | 60 Go |
| 102 | win-client | Windows Client | Poste utilisateur | 4 Go | 40 Go |

---

## VM 100 — Kali Linux

### Présentation
Kali Linux est la distribution de référence en cybersécurité offensive. Elle embarque plus de 600 outils préinstallés couvrant la reconnaissance, l'exploitation, la post-exploitation et le reporting.

### Configuration VM

| Paramètre | Valeur |
|-----------|--------|
| VM ID | 100 |
| OS | Kali Linux (dernière version) |
| CPU | 2 cœurs |
| RAM | 4 Go |
| Disque | 40 Go |
| Réseau | vmbr0 (192.168.10.x) |

### Outils utilisés dans le lab

| Catégorie | Outils |
|-----------|--------|
| Reconnaissance | `nmap`, `netdiscover`, `whois` |
| Scan de vulnérabilités | `nikto`, `OpenVAS` |
| Exploitation | `metasploit`, `searchsploit` |
| Mots de passe | `hydra`, `john`, `hashcat` |
| Réseau | `wireshark`, `tcpdump`, `ettercap` |
| Web | `burpsuite`, `sqlmap`, `dirb` |

### Cas d'usage dans le lab
- Scanner le réseau `192.168.10.x` pour découvrir les machines
- Attaquer Windows Server (VM 101) pour tester la sécurité de l'Active Directory
- Capturer et analyser le trafic réseau entre les VMs
- Pratiquer des scénarios CTF (Capture The Flag)

---

## VM 101 — Windows Server

### Présentation
Windows Server simule un serveur d'entreprise avec Active Directory. C'est l'environnement cible principal pour les exercices de pentest interne.

### Configuration VM

| Paramètre | Valeur |
|-----------|--------|
| VM ID | 101 |
| OS | Windows Server 2019/2022 |
| CPU | 2 cœurs |
| RAM | 4 Go |
| Disque | 60 Go |
| Réseau | vmbr0 (192.168.10.x) |

### Services configurés

| Service | Rôle |
|---------|------|
| Active Directory Domain Services (AD DS) | Gestion des utilisateurs et groupes |
| DNS Server | Résolution de noms dans le domaine |
| DHCP Server | Distribution d'IPs aux clients du domaine |

### Cas d'usage dans le lab
- Simuler un environnement Active Directory d'entreprise
- Pratiquer les attaques AD (Pass-the-Hash, Kerberoasting, BloodHound)
- Configurer et tester des GPO (Group Policy Objects)
- Analyser les logs de sécurité Windows (Event Viewer)

---

## VM 102 — Windows Client

### Présentation
Le poste client Windows simule un poste utilisateur standard, joint au domaine Active Directory. C'est la cible typique d'attaques de type phishing, élévation de privilèges ou mouvement latéral.

### Configuration VM

| Paramètre | Valeur |
|-----------|--------|
| VM ID | 102 |
| OS | Windows 10/11 |
| CPU | 2 cœurs |
| RAM | 4 Go |
| Disque | 40 Go |
| Réseau | vmbr0 (192.168.10.x) |

### Cas d'usage dans le lab
- Joint au domaine Active Directory (VM 101)
- Cible d'attaques depuis Kali (VM 100)
- Tester des scénarios de compromission de poste utilisateur
- Pratiquer la détection et la remédiation d'incidents

---

## Gestion des snapshots

Les snapshots permettent de **sauvegarder l'état d'une VM** à un instant T et d'y revenir en cas de problème. C'est essentiel dans un lab de cybersécurité où les VMs peuvent être endommagées lors d'exercices.

### Créer un snapshot
1. Sélectionner la VM dans Proxmox
2. Onglet **Snapshots** → **Take Snapshot**
3. Donner un nom explicite (ex: `avant-pentest-2024-05`, `AD-config-ok`)
4. Cocher **Include RAM** si la VM doit être restaurée dans le même état mémoire

### Restaurer un snapshot
1. Onglet **Snapshots** → sélectionner le snapshot
2. Cliquer **Rollback**
3. La VM revient à l'état exact au moment du snapshot

### Bonnes pratiques snapshots
- ✅ Créer un snapshot **avant chaque exercice offensif**
- ✅ Nommer les snapshots avec une date et un contexte
- ✅ Conserver un snapshot "propre" de chaque VM (état initial configuré)
- ⚠️ Les snapshots consomment de l'espace disque — nettoyer régulièrement les anciens

---

## Bonnes pratiques appliquées

### Sécurité d'accès
- Interface Proxmox accessible **uniquement via LAN ou VPN** — jamais exposée sur Internet
- Authentification par mot de passe fort sur le compte `root`
- Accès web en HTTPS (certificat auto-signé)

### Isolation des VMs
- Toutes les VMs sont sur `vmbr0` et passent par pfSense
- Possibilité de créer un bridge isolé (`vmbr1`) pour des exercices sans accès réseau externe

### Gestion des ressources
- RAM allouée dynamiquement selon les besoins
- Snapshots avant chaque exercice sensible
- Nettoyage régulier des snapshots obsolètes

---

## Commandes utiles

### Depuis l'interface web Proxmox

| Action | Chemin |
|--------|--------|
| Démarrer une VM | Sélectionner VM → Start |
| Accéder à la console | Sélectionner VM → Console |
| Créer un snapshot | Sélectionner VM → Snapshots → Take Snapshot |
| Voir l'utilisation des ressources | Datacenter → Summary |
| Gérer le stockage | Datacenter → Storage |

### Depuis le terminal Proxmox (SSH ou console)

```bash
# Lister toutes les VMs et leur état
qm list

# Démarrer une VM
qm start 100

# Arrêter une VM
qm stop 100

# Redémarrer une VM
qm reboot 101

# Voir les infos d'une VM
qm config 100

# Créer un snapshot en ligne de commande
qm snapshot 100 nom-du-snapshot

# Lister les snapshots d'une VM
qm listsnapshot 100

# Restaurer un snapshot
qm rollback 100 nom-du-snapshot

# Voir l'utilisation des ressources du nœud
pvesh get /nodes/proxmox-lab/status
```
