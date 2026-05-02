# 🏗️ Architecture du Cyber Lab

## Sommaire
- [Vue d'ensemble](#vue-densemble)
- [Choix techniques](#choix-techniques)
- [Matériel](#matériel)
- [Schéma réseau détaillé](#schéma-réseau-détaillé)
- [Plan d'adressage IP](#plan-dadressage-ip)
- [Flux réseau](#flux-réseau)
- [Sécurité de l'infrastructure](#sécurité-de-linfrastructure)

---

## Vue d'ensemble

Ce cyber lab personnel a été conçu pour pratiquer l'administration réseau, la sécurisation d'infrastructure et les techniques offensives/défensives dans un environnement **totalement isolé** du réseau domestique.

L'objectif est de reproduire à petite échelle une infrastructure d'entreprise réelle, avec :
- Un **firewall dédié** pour la segmentation et le filtrage
- Un **hyperviseur bare-metal** pour la virtualisation
- Plusieurs **VMs spécialisées** (attaque, serveur, client)
- Un **accès distant sécurisé** via VPN

---

## Choix techniques

### Pourquoi pfSense comme firewall ?
- Solution open source professionnelle, utilisée en entreprise
- Interface web complète et intuitive
- Supporte WireGuard, OpenVPN, IPSec nativement
- Gestion avancée des règles firewall, NAT, VLAN
- Fonctionne sur du matériel recyclé (ThinkPad T430)

### Pourquoi Proxmox comme hyperviseur ?
- Hyperviseur bare-metal open source de niveau professionnel
- Interface web pour gérer toutes les VMs
- Supporte KVM (machines virtuelles) et LXC (conteneurs)
- Snapshots, sauvegardes, clonage de VMs intégrés
- Très utilisé en entreprise et dans les homelab

### Pourquoi WireGuard comme VPN ?
- Protocole moderne, rapide et simple à configurer
- Chiffrement de pointe (ChaCha20, Curve25519, BLAKE2)
- Intégré nativement dans le kernel Linux
- Faible consommation de ressources
- Multi-clients (iPhone + laptop simultanément)

### Pourquoi ce choix de VMs ?
| VM | Justification |
|----|--------------|
| Kali Linux | Distribution de référence en pentest, outils offensifs intégrés |
| Windows Server | Simuler un environnement Active Directory d'entreprise |
| Windows Client | Simuler un poste utilisateur, cible d'attaques |

---

## Matériel

> 💡 **Tous les PC de ce lab étaient sous Windows à l'origine.** L'intégralité des OS a été remplacé manuellement : création des clés USB bootables, modification du BIOS/UEFI, installation from scratch de chaque système. Le Lenovo T430 a également été démonté et upgradé matériellement avant déploiement.

### Firewall — Lenovo ThinkPad T430

| Composant | Détail |
|-----------|--------|
| Rôle | Firewall / Routeur / Serveur VPN |
| OS | pfSense CE (FreeBSD) — remplace Windows d'origine |
| CPU | Intel Core i5-3320M |
| RAM | 16 Go DDR3 — **upgrade manuel** (était 8 Go d'origine) |
| Stockage | SSD 240 Go Kingston — **ajout manuel** (pas de SSD d'origine) |
| Réseau | Interface interne (WAN vers box FAI) + adaptateur USB-Ethernet (LAN vers switch) |

> 🔧 **Intervention matérielle :** démontage complet du ThinkPad T430, remplacement du module RAM et installation d'un SSD Kingston 240 Go. Le laptop ne disposait d'aucun SSD à l'origine. Tout le matériel a été installé et configuré manuellement.

> 💡 L'interface réseau interne du T430 sert d'interface WAN (connexion vers la box FAI). L'adaptateur USB-Ethernet sert de LAN vers le switch.

### Hyperviseur — PC Ryzen 7

| Composant | Détail |
|-----------|--------|
| Rôle | Hyperviseur bare-metal |
| OS | Proxmox VE 9.1 — remplace Windows d'origine |
| CPU | AMD Ryzen 7 |
| RAM | 32 Go DDR4 |
| Stockage | SSD 1 To |
| Réseau | Connecté au switch LAN |

### PC Principal — Ryzen 5

| Composant | Détail |
|-----------|--------|
| Rôle | Poste d'administration |
| OS | Ubuntu — remplace Windows d'origine |
| CPU | AMD Ryzen 5 |
| RAM | 16 Go DDR4 |
| Stockage | SSD 512 Go |
| Réseau | Connecté au LAN via switch |
| Accès | Direct vers Proxmox et pfSense sans VPN |

### Laptop HP — Client distant

| Composant | Détail |
|-----------|--------|
| Rôle | Accès distant depuis l'école |
| OS | Linux Mint — remplace Windows d'origine |
| RAM | 16 Go |
| Stockage | SSD 512 Go |
| VPN | WireGuard (peer 10.0.0.x) |
| Réseau école | Filtré via portail ALCASAR |

---

## Schéma réseau détaillé

```
                        INTERNET
                            │
                    ┌───────▼───────┐
                    │   Box FAI     │
                    │  DHCP / NAT   │
                    │ 192.168.1.0/24│
                    │               │
                    │ Ports ouverts │ ← redirection vers pfSense WAN
                    │  UDP 51820    │   (WireGuard)
                    └───────┬───────┘
                            │ WAN — tout le trafic entrant passe ici
                    ┌───────▼───────┐
                    │   pfSense     │  Lenovo T430 (upgradé)
                    │               │
                    │  WAN 192.168.1.x  ← reçoit le trafic de la box
                    │  LAN 192.168.10.x ← route vers le switch
                    │  VPN 10.0.0.x │
                    │               │
                    │  Firewall     │  ← tout le trafic est filtré ici
                    │  NAT          │  ← translation d'adresses
                    │  WireGuard    │  ← serveur VPN
                    └───────┬───────┘
                            │ LAN — tout passe par pfSense avant d'arriver ici
                       ┌────▼────┐
                       │ Switch  │ ← point de distribution LAN
                       └────┬────┘
                            │
               ┌────────────┼─────────────┐
               │            │             │
       ┌───────▼──────┐     │    ┌────────▼───────┐
       │  PC Ryzen 5  │     │    │  Proxmox VE 9.1│  Ryzen 7
       │  Ubuntu      │     │    │  192.168.10.x  │
       │  192.168.10.x│     │    │                │
       │  Poste admin │     │    │  ┌───────────┐ │
       └──────────────┘     │    │  │ VM 100    │ │
                            │    │  │ Kali Linux│ │
                            │    │  └───────────┘ │
                            │    │  ┌───────────┐ │
                            │    │  │ VM 101    │ │
                            │    │  │ Win Server│ │
                            │    │  └───────────┘ │
                            │    │  ┌───────────┐ │
                            │    │  │ VM 102    │ │
                            │    │  │ Win Client│ │
                            │    │  └───────────┘ │
                            │    └────────────────┘
                            │
                     (autres appareils
                      branchés au switch)

Accès distant (WireGuard) :
  📱 iPhone    ── UDP 51820 ──► Box FAI ──► pfSense ──► Switch ──► Lab
  💻 Laptop HP ── UDP 51820 ──► Box FAI ──► pfSense ──► Switch ──► Lab
  🖥️  Ryzen 5  ── LAN direct ────────────────────────► Switch ──► Lab
```

> ⚠️ **Point clé :** Tout le trafic entrant et sortant du lab passe obligatoirement par pfSense. Le switch ne fait que distribuer le signal — c'est pfSense qui décide ce qui est autorisé ou non.

---

## Plan d'adressage IP

### Réseau domestique (box FAI)
| Hôte | IP | Rôle |
|------|----|------|
| Box FAI | 192.168.1.1 | Passerelle Internet |
| pfSense WAN | 192.168.1.x | Interface WAN firewall |

### Réseau LAN lab (isolé)
| Hôte | IP | Rôle |
|------|----|------|
| pfSense LAN | 192.168.10.x | Passerelle du lab |
| Proxmox | 192.168.10.x | Hyperviseur |
| PC Ryzen 5 | 192.168.10.x | Poste admin |

### Réseau VPN WireGuard
| Hôte | IP | Rôle |
|------|----|------|
| pfSense (serveur) | 10.0.0.x | Serveur WireGuard |
| iPhone | 10.0.0.x | Peer mobile |
| Laptop HP | 10.0.0.x | Peer école |

> 💡 Les IPs exactes sont volontairement masquées dans cette documentation publique. C'est une bonne pratique de ne pas exposer son plan d'adressage réel.

---

## Flux réseau

### Accès normal depuis le LAN
```
PC Ryzen 5 → Switch → pfSense LAN → Proxmox / VMs
```
Accès direct, pas de VPN nécessaire.

### Accès distant via VPN
```
Laptop HP / iPhone
    │
    │ UDP 51820 chiffré (WireGuard)
    ▼
Box FAI (NAT)
    │
    ▼
pfSense WAN → déchiffrement → tun_wg0
    │
    │ Règle firewall : WireGuard → LAN autorisé
    ▼
Proxmox / VMs (192.168.10.x)
```

### Trafic entre VMs (lab interne)
```
VM Kali (attaquant)
    │
    │ Réseau virtuel Proxmox
    ▼
VM Windows Server / Client (cible)
```
Les VMs communiquent entre elles via le réseau virtuel de Proxmox, sans passer par pfSense. Cela permet de simuler des scénarios d'attaque interne.

---

## Sécurité de l'infrastructure

### Isolation réseau
- Le réseau lab `192.168.10.x` est **séparé** du réseau domestique `192.168.1.x`
- pfSense contrôle **tout le trafic** entre les deux réseaux
- Sans VPN actif, le lab est **inaccessible depuis l'extérieur**

### Accès VPN
- Chaque peer WireGuard a sa propre **paire de clés unique**
- Les clés privées ne transitent **jamais sur le réseau**
- Le trafic est **chiffré de bout en bout** (ChaCha20-Poly1305)
- Le split tunneling est activé : seul le trafic vers le lab passe par le VPN

### Bonnes pratiques appliquées
- ✅ Pas de clés ou mots de passe dans les fichiers versionnés
- ✅ IPs réelles masquées dans la documentation publique
- ✅ Permissions restrictives sur les fichiers de config (`chmod 600`)
- ✅ Firewall avec politique de refus par défaut
- ✅ Accès Proxmox uniquement via VPN ou LAN (pas exposé sur Internet)
