# 🔐 Configuration VPN WireGuard

## Sommaire
- [Pourquoi WireGuard ?](#pourquoi-wireguard-)
- [Architecture VPN](#architecture-vpn)
- [Configuration côté serveur (pfSense)](#configuration-côté-serveur-pfsense)
- [Configuration côté client Linux (Laptop HP)](#configuration-côté-client-linux-laptop-hp)
- [Configuration côté client mobile (iPhone)](#configuration-côté-client-mobile-iphone)
- [Règles Firewall associées](#règles-firewall-associées)
- [Tests et vérifications](#tests-et-vérifications)
- [Dépannage](#dépannage)

---

## Pourquoi WireGuard ?

WireGuard est un protocole VPN moderne qui présente plusieurs avantages par rapport aux solutions traditionnelles comme OpenVPN ou IPSec :

| Critère | WireGuard | OpenVPN |
|---------|-----------|---------|
| Performance | ⚡ Très rapide (intégré au kernel Linux) | 🐢 Plus lent |
| Configuration | ✅ Simple (quelques lignes) | ❌ Complexe |
| Sécurité | ✅ Cryptographie moderne (ChaCha20, Curve25519) | ✅ Solide mais vieillissant |
| Consommation batterie | ✅ Faible (keepalive efficace) | ❌ Plus élevée |
| Taille du code | ~4 000 lignes | ~600 000 lignes |

Dans le cadre de ce lab, WireGuard permet d'accéder à l'infrastructure depuis l'école ou en mobilité de façon sécurisée et transparente.

---

## Architecture VPN

```
┌─────────────────────────────────────────────────────┐
│                     Internet                        │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
    ┌──────▼──────┐            ┌──────▼──────┐
    │  iPhone     │            │  Laptop HP  │
    │  10.0.0.x   │            │  10.0.0.x   │
    │  (mobile)   │            │  Linux Mint │
    └──────┬──────┘            └──────┬──────┘
           │    WireGuard UDP 51820   │
           └──────────┬───────────────┘
                      │
             ┌────────▼────────┐
             │  pfSense        │
             │  tun_wg0        │
             │  10.0.0.x/24    │
             └────────┬────────┘
                      │
             ┌────────▼────────┐
             │  LAN Lab        │
             │  192.168.10.x   │
             │  Proxmox + VMs  │
             └─────────────────┘
```

**Principe de fonctionnement :**
- pfSense fait office de **serveur WireGuard** (écoute en permanence)
- Chaque appareil distant est un **peer** (client) avec sa propre paire de clés
- La communication est **chiffrée de bout en bout** avec des clés asymétriques
- Chaque peer a une **IP dédiée** dans le tunnel (`10.0.0.x/32`)

---

## Configuration côté serveur (pfSense)

### Prérequis
- pfSense CE installé et fonctionnel
- Package WireGuard installé via **System → Package Manager**
- Redirection de port configurée sur la box FAI : `UDP 51820 → IP WAN pfSense`

### Création du tunnel

**VPN → WireGuard → Tunnels → Add Tunnel**

| Paramètre | Valeur |
|-----------|--------|
| Enable | ✅ Coché |
| Description | VPN-LAB |
| Listen Port | 51820 |
| Interface Address | 10.0.0.x/24 |

> ⚠️ Cliquer sur **Generate** pour créer la paire de clés du serveur. Ne jamais partager la clé privée.

### Ajout des peers

**VPN → WireGuard → Peers → Add Peer**

Pour chaque appareil client (iPhone, laptop...) :

| Paramètre | Valeur |
|-----------|--------|
| Enable | ✅ Coché |
| Tunnel | tun_wg0 (VPN-LAB) |
| Description | Nom de l'appareil |
| Dynamic Endpoint | ✅ Coché (IP du client peut changer) |
| Keep Alive | 25 |
| Public Key | Clé publique générée côté client |
| Allowed IPs | 10.0.0.x/32 (IP dédiée du peer) |

### Assignation de l'interface

**Interfaces → Assignments → ajouter tun_wg0**

Puis activer l'interface dans **Interfaces → WireGuard** :
- Enable : ✅
- Description : WireGuard

### Règles firewall WireGuard

**Firewall → Rules → WireGuard**

Ajouter une règle :

| Paramètre | Valeur |
|-----------|--------|
| Action | Pass |
| Protocol | Any |
| Source | WireGuard net |
| Destination | LAN net (192.168.10.x/24) |
| Description | Autoriser VPN vers LAN |

---

## Configuration côté client Linux (Laptop HP)

### Environnement
- OS : Linux Mint
- Réseau école : filtré via portail ALCASAR
- Connectivité : UDP 51820 autorisé (testé et validé)

### Installation

```bash
sudo apt update && sudo apt install wireguard -y
```

### Génération des clés

```bash
# Générer la paire de clés
wg genkey | tee privatekey | wg pubkey > publickey

# Afficher la clé privée (à garder secrète)
cat privatekey

# Afficher la clé publique (à donner à pfSense)
cat publickey
```

> ⚠️ La clé publique est celle qu'on renseigne dans pfSense côté peer.  
> ⚠️ La clé privée ne doit jamais être partagée ni commitée sur GitHub.

### Fichier de configuration

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = <clé_privée_du_client>
Address = 10.0.0.x/32
DNS = 10.0.0.x

[Peer]
PublicKey = <clé_publique_du_serveur_pfsense>
Endpoint = <IP_publique_box>:51820
AllowedIPs = 192.168.10.0/24, 10.0.0.0/24
PersistentKeepalive = 25
```

**Explication des paramètres :**

| Paramètre | Rôle |
|-----------|------|
| `Address` | IP du client dans le tunnel VPN |
| `DNS` | Résolution DNS via pfSense |
| `PublicKey` | Clé publique du serveur (pfSense) |
| `Endpoint` | IP publique de la box + port WireGuard |
| `AllowedIPs` | Réseaux accessibles via le tunnel |
| `PersistentKeepalive` | Maintient la connexion active (utile derrière NAT) |

> ⚠️ `AllowedIPs` ne contient **pas** `0.0.0.0/0` — seul le trafic vers le lab passe par le VPN, le reste (navigation web) passe par la connexion normale. C'est ce qu'on appelle le **split tunneling**.

### Sécurisation du fichier

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

### Commandes utiles

```bash
# Démarrer le VPN
sudo wg-quick up wg0

# Arrêter le VPN
sudo wg-quick down wg0

# Vérifier l'état de la connexion
sudo wg show

# Démarrage automatique au boot
sudo systemctl enable wg-quick@wg0

# Désactiver le démarrage automatique
sudo systemctl disable wg-quick@wg0
```

### Vérifier que la connexion fonctionne

```bash
sudo wg show
```

Une connexion active ressemble à :
```
interface: wg0
  public key: <clé_publique_client>
  private key: (hidden)
  listening port: XXXXX

peer: <clé_publique_serveur>
  endpoint: <IP_publique>:51820
  allowed ips: 192.168.10.0/24, 10.0.0.0/24
  latest handshake: 5 seconds ago        ← ✅ tunnel actif
  transfer: 2.51 KiB received, 1.94 KiB sent
  persistent keepalive: every 25 seconds
```

> ✅ Si `latest handshake` affiche un temps récent → le tunnel est actif.  
> ❌ Si `latest handshake` n'apparaît pas → problème de connectivité.

---

## Configuration côté client mobile (iPhone)

### Application
Installer **WireGuard** depuis l'App Store (application officielle).

### Création du tunnel
1. Ouvrir l'app → **+** → **Créer depuis zéro**
2. Renseigner les mêmes paramètres que pour Linux (adapter l'IP du peer)
3. Activer le tunnel depuis l'app

### Partage de la clé publique vers pfSense
- Dans l'app, afficher les détails du tunnel
- Copier la **clé publique** et la renseigner dans le peer pfSense correspondant

---

## Règles Firewall associées

### Sur la box FAI
Redirection de port nécessaire :

| Protocole | Port externe | Destination | Port interne |
|-----------|-------------|-------------|--------------|
| UDP | 51820 | IP WAN pfSense | 51820 |

### Sur pfSense — Interface WAN
Autoriser le trafic entrant WireGuard :

| Action | Protocol | Port dest | Description |
|--------|----------|-----------|-------------|
| Pass | UDP | 51820 | WireGuard entrant |

---

## Tests et vérifications

### Checklist de validation

```
[ ] sudo wg show affiche "latest handshake" récent
[ ] ping 10.0.0.x (pfSense côté tunnel) répond
[ ] ping 192.168.10.x (Proxmox) répond
[ ] https://192.168.10.x:8006 accessible dans le navigateur
[ ] iPhone toujours fonctionnel après ajout du nouveau peer
```

### Tester la connectivité réseau

```bash
# Ping pfSense via tunnel
ping 10.0.0.x

# Ping Proxmox
ping 192.168.10.x

# Accès Proxmox web
curl -k https://192.168.10.x:8006
```

---

## Dépannage

### Le handshake n'apparaît pas

**Causes possibles et solutions :**

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| Pas de handshake | Port UDP 51820 bloqué | Tester depuis 4G, changer le port |
| Pas de handshake | Mauvaise clé publique | Vérifier les clés dans pfSense |
| Pas de handshake | Redirection de port absente | Vérifier le NAT sur la box |
| Handshake OK mais pas de ping | Règle firewall manquante | Vérifier les règles WireGuard → LAN |
| IP en IPv6 sur le client | Le client utilise IPv6 | Utiliser l'IP publique IPv4 dans Endpoint |

### Le réseau de l'école bloque UDP 51820

Si le port est bloqué, solutions par ordre de complexité :

1. **Changer le port** vers UDP 443 (souvent autorisé)
2. **Encapsuler dans WebSocket** via `wstunnel` (WireGuard over TCP 443)

### Vérifier les logs pfSense

**Status → System Logs → Firewall** — filtrer par port 51820 pour voir si les paquets arrivent et sont acceptés.
