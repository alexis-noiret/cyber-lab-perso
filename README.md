# 🛡️ Cyber Lab Personnel — Infrastructure Sécurisée

![Status](https://img.shields.io/badge/Status-Opérationnel-brightgreen)
![pfSense](https://img.shields.io/badge/Firewall-pfSense-blue)
![Proxmox](https://img.shields.io/badge/Hyperviseur-Proxmox_9.1-orange)
![WireGuard](https://img.shields.io/badge/VPN-WireGuard-88171A)
![Linux](https://img.shields.io/badge/OS-Linux-yellow)

Mise en place d'un cyber lab personnel isolé avec firewall dédié, hyperviseur bare-metal, machines virtuelles et accès VPN sécurisé depuis l'extérieur. Projet réalisé dans le cadre de ma formation en cybersécurité pour pratiquer l'administration réseau, la virtualisation et la sécurisation d'infrastructure.

---

## 🗺️ Architecture générale

```
Internet
    │
    │ IP publique (box)
    │
┌───▼────────────────────┐
│  Box FAI               │
│  192.168.1.0/24        │
│  NAT → pfSense :51820  │
└───────────┬────────────┘
            │ WAN 192.168.1.169
┌───────────▼────────────┐
│  Firewall pfSense      │  ← Lenovo ThinkPad T430
│  LAN 192.168.10.1/24   │
│  VPN WireGuard tun_wg0 │
│  10.0.0.1/24           │
└───────────┬────────────┘
            │ LAN via switch
┌───────────▼────────────┐
│  Proxmox VE 9.1        │  ← Ryzen 7 (bare-metal)
│  192.168.10.50         │
│  ┌─────────────────┐   │
│  │ VM 100 - Kali   │   │
│  │ VM 101 - WinSrv │   │
│  │ VM 102 - WinClt │   │
│  └─────────────────┘   │
└────────────────────────┘

Accès distant via WireGuard :
  📱 iPhone      → 10.0.0.2/32
  💻 Laptop HP   → 10.0.0.3/32  (Linux Mint, depuis l'école)
  🖥️ PC principal → 192.168.10.x (accès LAN direct)
```

---

## 🧰 Matériel utilisé

| Rôle | Machine | Specs |
|------|---------|-------|
| Firewall | Lenovo ThinkPad T430 | pfSense CE, 2 interfaces réseau (interne + USB-Ethernet) |
| Hyperviseur | PC Ryzen 7 | Proxmox VE 9.1, connecté via switch au LAN |
| PC principal | PC Ryzen 5 | lINUX Ubuntu | Accès direct au lab via LAN |
| Client distant | Laptop HP | Linux Mint, accès VPN WireGuard depuis l'école |
| Client mobile | iPhone 17 | Accès VPN WireGuard depuis l'extérieur |

---

## 🔥 Firewall — pfSense

**Rôle :** Segmentation réseau, NAT, filtrage du trafic, serveur VPN WireGuard.

**Configuration réseau :**
- Interface WAN : `192.168.1.169/24` (vers la box FAI)
- Interface LAN : `192.168.10.1/24` (vers le switch et Proxmox)
- Tunnel WireGuard : `10.0.0.1/24`

**Règles firewall :**
- Trafic WireGuard → LAN autorisé pour les peers déclarés
- Isolation du lab : le réseau `192.168.10.0/24` n'est pas accessible sans VPN depuis l'extérieur

📄 [Documentation pfSense détaillée](docs/02-pfsense.md)

---

## 🖥️ Hyperviseur — Proxmox VE 9.1

**Rôle :** Hébergement des machines virtuelles du lab.

**Machines virtuelles :**

| VM | OS | Rôle |
|----|-----|------|
| VM 100 | Kali Linux | Pentest, reconnaissance, outils offensifs |
| VM 101 | Windows Server | Active Directory, services réseau |
| VM 102 | Windows Client | Poste utilisateur, tests d'exploitation |

**Accès :** `https://192.168.10.50:8006` (via VPN ou LAN)

📄 [Documentation Proxmox détaillée](docs/03-proxmox.md)

---

## 🔐 VPN — WireGuard

**Rôle :** Accès sécurisé au lab depuis l'extérieur (école, mobilité).

**Configuration :**
- Protocole : WireGuard (UDP 51820)
- Serveur : pfSense `10.0.0.1`
- Peers configurés :
  - iPhone (`10.0.0.2`) — accès mobile
  - Laptop HP Linux Mint (`10.0.0.3`) — accès depuis l'école via réseau ALCASAR

**Réseau VPN :** `10.0.0.0/24`  
**Routes accessibles via VPN :** `192.168.10.0/24`, `10.0.0.0/24`

📄 [Documentation WireGuard détaillée](docs/04-wireguard.md)

---

## 🎯 Objectifs du projet

- ✅ Mettre en place un réseau isolé avec firewall dédié
- ✅ Déployer un hyperviseur bare-metal avec plusieurs VMs
- ✅ Configurer un accès VPN sécurisé multi-clients
- ✅ Accéder au lab depuis l'école via réseau filtré (ALCASAR)
- 🔄 Documenter les configurations pour reproductibilité
- 🔜 Mettre en place un SIEM (ELK) pour la supervision des logs
- 🔜 Pratiquer des scénarios d'attaque/défense entre les VMs

---

## 📚 Compétences mises en pratique

`Administration réseau` `pfSense` `Proxmox` `Virtualisation` `WireGuard` `VPN` `Firewall` `NAT` `Segmentation réseau` `Linux` `Windows Server` `Kali Linux`

---

## 📁 Structure du repo

```
cyber-lab-perso/
├── README.md                  ← Vue d'ensemble (ce fichier)
├── docs/
│   ├── 01-architecture.md     ← Schéma et choix techniques détaillés
│   ├── 02-pfsense.md          ← Configuration pfSense
│   ├── 03-proxmox.md          ← Configuration Proxmox et VMs
│   └── 04-wireguard.md        ← Configuration VPN WireGuard
└── diagrams/
    └── network-schema.png     ← Schéma réseau visuel
```

---

## ⚠️ Avertissement

Ce lab est utilisé dans un cadre strictement éducatif et personnel. Le réseau est isolé et les tests sont réalisés uniquement sur des machines m'appartenant. Aucune activité n'est dirigée vers des systèmes tiers.

---

## 👤 Auteur

**Alexis Noiret** — Étudiant en Bachelor Cybersécurité  
🌐 [Portfolio](https://alexis-noiret.students-laplateforme.io/)  
🐙 [GitHub](https://github.com/alexis-noiret)  
💼 [LinkedIn](https://linkedin.com/in/alexis-noiret-726019349)  
📧 alexis.noiret@laplateforme.io

*Recherche d'alternance en cybersécurité — Octobre 2026*
