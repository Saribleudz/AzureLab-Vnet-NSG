# Azure Network Lab — VNet, Subnets & NSGs

> Lab pratique pour s'exercer aux fondamentaux réseau sur Azure : segmentation d'un réseau virtuel, règles firewall avec les Network Security Groups, et communication entre VMs via le réseau privé.

---

## Architecture

```
                    Internet
                       │
                  SSH · HTTP(80)
                       │
       ┌───────────────▼────────────────────────────────┐
       │          vm-web-vnet — 10.0.0.0/16             │
       │              West Europe                        │
       │                                                 │
       │  ┌──────────────────┐  ┌──────────────────┐   │
       │  │  Subnet-web      │  │  Subnet-db        │   │
       │  │  10.0.1.0/24     │  │  10.0.2.0/24      │   │
       │  │                  │  │                   │   │
       │  │  NSG-web-v2      │  │  NSG-db-v2        │   │
       │  │  ✓ SSH(22) IN    │  │  ✓ SSH from web   │   │
       │  │  ✓ HTTP(80) IN   │  │  ✗ tout internet  │   │
       │  │  ✗ tout le reste │  │  ✗ ICMP           │   │
       │  │                  │  │                   │   │
       │  │  ┌────────────┐  │  │  ┌────────────┐  │   │
       │  │  │  VM-web    │◄─┼──┼─►│  VM-db     │  │   │
       │  │  │ 10.0.1.4   │  │  │  │ 10.0.2.4   │  │   │
       │  │  │ D2sv3      │  │  │  │ D2sv3      │  │   │
       │  │  └────────────┘  │  │  └────────────┘  │   │
       │  └──────────────────┘  └──────────────────┘   │
       └─────────────────────────────────────────────────┘
                     rg-lab-reseau
```

---

## Stack

| Ressource | Valeur |
|---|---|
| Abonnement | Azure subscription 1 (Free Trial) |
| Resource group | `rg-lab-reseau` |
| Région | **West Europe** (voir note ci-dessous) |
| VNet | `vm-web-vnet` — 10.0.0.0/16 |
| Subnet-web | 10.0.1.0/24 |
| Subnet-db | 10.0.2.0/24 |
| VM-web | Ubuntu 24.04 LTS, Standard D2s_v3, IP publique `172.201.105.88` |
| VM-db | Ubuntu 24.04 LTS, Standard D2s_v3, pas d'IP publique |
| NSG-web-v2 | SSH + HTTP autorisés en entrée, tout le reste bloqué |
| NSG-db-v2 | SSH depuis Subnet-web uniquement, tout le reste bloqué dont ICMP |

---

## Prérequis

- Un compte Azure avec un abonnement actif
- Accès au portail Azure
- Bases en réseau (subnets, CIDR, règles firewall)
- Terminal Linux avec SSH (WSL sur Windows fonctionne)

---

## Notes de déploiement — ce qui s'est vraiment passé

### Région : West Europe au lieu de France Central

Le plan initial était de tout déployer en **France Central**. Cependant, l'abonnement Free Trial n'avait aucun quota disponible pour les VM B-series dans cette région (erreur `NotAvailableForSubscription`). Après avoir testé plusieurs régions, **West Europe** (Amsterdam) était la seule région où le Standard_D2s_v3 était disponible sous cet abonnement.

En conséquence, les NSGs initialement créés en France Central ne pouvaient pas être associés aux subnets en West Europe — Azure impose que les NSGs et les VNets soient dans la même région. Les NSGs ont donc été recréés en West Europe sous les noms `nsg-web-v2` et `nsg-db-v2`.

**Leçon retenue :** toujours vérifier la disponibilité des tailles de VM dans la région cible avant de concevoir l'architecture, surtout sur les abonnements Free Trial où les quotas sont limités.

### VNet : créé automatiquement par Azure lors du déploiement de VM-web

Plutôt que de créer un VNet standalone en premier, Azure a auto-créé `vm-web-vnet` (10.0.0.0/16) au moment du déploiement de VM-web. Un subnet `default` (10.0.0.0/24) a également été créé automatiquement et ne pouvait pas être supprimé car VM-web y était attachée. `Subnet-web` (10.0.1.0/24) et `Subnet-db` (10.0.2.0/24) ont été ajoutés manuellement ensuite.

### Taille de VM : Standard_D2s_v3 au lieu de Standard_B1s

Le Standard_B1s est la taille recommandée pour ce type de lab (1 vCPU, 1 GiB RAM, éligible free tier). Il était indisponible dans toutes les régions testées sous cet abonnement Free Trial. Le Standard_D2s_v3 (2 vCPUs, 8 GiB RAM, ~0,12$/h) a été utilisé à la place. Coût total du lab : moins de 0,25$.

---

## Déploiement — étape par étape

### 1. Créer le Resource Group

**Portail Azure** → *Resource groups* → **+ Create**

| Champ | Valeur |
|---|---|
| Abonnement | Azure subscription 1 |
| Resource group | `rg-lab-reseau` |
| Région | West Europe |

---

### 2. Créer le réseau virtuel

**Portail** → *Virtual networks* → **+ Create**

**Onglet Basics :**
- Resource group : `rg-lab-reseau`
- Nom : `vm-web-vnet`
- Région : West Europe

**Onglet IP Addresses :**
- Address space : `10.0.0.0/16`
- Ajouter **Subnet-web** : `10.0.1.0/24`
- Ajouter **Subnet-db** : `10.0.2.0/24`

> 📸 `02-vnet-subnets.png` — liste des subnets avec Subnet-web et Subnet-db visibles.

---

### 3. Créer les Network Security Groups

#### NSG-web-v2

**Portail** → *Network security groups* → **+ Create**
- Nom : `nsg-web-v2`
- Resource group : `rg-lab-reseau`
- Région : West Europe

**Règles inbound à ajouter :**

| Priorité | Nom | Port | Protocole | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-SSH | 22 | TCP | Any | Allow |
| 110 | Allow-HTTP | 80 | TCP | Any | Allow |
| 4096 | Deny-All-Inbound | * | Any | Any | Deny |

> La règle `Deny-All-Inbound` à priorité 4096 court-circuite la règle par défaut `AllowVnetInBound` (65000). Azure affiche un warning indiquant que cela bloque le trafic VNet interne — c'est intentionnel.

#### NSG-db-v2

**Portail** → *Network security groups* → **+ Create**
- Nom : `nsg-db-v2`
- Région : West Europe

**Règles inbound :**

| Priorité | Nom | Port | Protocole | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-SSH-From-Web | 22 | TCP | 10.0.1.0/24 | Allow |
| 4096 | Deny-All-Inbound | * | Any | Any | Deny |

> `nsg-db-v2` bloque tout sauf le SSH depuis Subnet-web. Cela inclut l'ICMP (ping) — voir les tests de connectivité ci-dessous.

> 📸 `03-nsg-web-rules.png` · `04-nsg-db-rules.png`

---

### 4. Associer les NSGs aux subnets

**Ressource VNet** → *Subnets* → cliquer sur chaque subnet → renseigner le champ Network security group :

| NSG | Subnet |
|---|---|
| `nsg-web-v2` | `Subnet-web` |
| `nsg-db-v2` | `Subnet-db` |

> 📸 `05-nsg-associations.png`

---

### 5. Déployer les VMs

#### VM-web

**Portail** → *Virtual machines* → **+ Create**

**Basics :**
- Nom : `vm-web`
- Région : West Europe
- Image : Ubuntu Server 24.04 LTS — x64 Gen2
- Taille : Standard_D2s_v3
- Authentification : SSH public key — télécharger le `.pem` immédiatement, Azure ne le stocke pas

**Networking :**
- VNet : `vm-web-vnet`
- Subnet : `Subnet-web`
- IP publique : créer nouvelle
- NIC NSG : **None** (le NSG est déjà appliqué au niveau du subnet)

#### VM-db

Même configuration, avec :
- Nom : `vm-db`
- Subnet : `Subnet-db`
- IP publique : **None**
- NIC NSG : **None**

> 📸 `06-vms-running.png` — les deux VMs en statut Running avec leurs IPs privées.

---

## Tests de connectivité

### 1. SSH sur VM-web depuis la machine locale

Le fichier `.pem` doit être copié dans le filesystem Linux avant d'appliquer les permissions (sous WSL, `chmod` ne fonctionne pas sur les chemins Windows montés en `/mnt/c/`) :

```bash
cp /mnt/c/Users/<user>/chemin/vm-web-key.pem ~/vm-web-key.pem
chmod 400 ~/vm-web-key.pem
ssh -i ~/vm-web-key.pem azureuser@172.201.105.88
```

> 📸 `07-ssh-vm-web.png`

---

### 2. Ping VM-db depuis VM-web

```bash
ping 10.0.2.4 -c 4
```

**Résultat : 100% packet loss** — attendu et correct.

NSG-db-v2 n'autorise que le port 22 TCP depuis `10.0.1.0/24`. Le ping utilise le protocole ICMP, qui n'est ni TCP ni UDP. La règle `Deny-All-Inbound` à priorité 4096 bloque tout le trafic ICMP. Ce résultat confirme que le NSG applique bien la politique d'isolation.

> 📸 `08-ping-vm-db.png`

---

### 3. SSH jump de VM-web vers VM-db

VM-db n'a pas d'IP publique et est injoignable depuis internet. Le seul moyen d'y accéder est via VM-web sur le réseau privé. Cela nécessite de copier la clé VM-db sur VM-web au préalable :

```bash
# Depuis la machine locale — copier la clé VM-db sur VM-web
scp -i ~/vm-web-key.pem ~/vm-db-key.pem azureuser@172.201.105.88:~/vm-db-key.pem

# Depuis VM-web — connexion à VM-db via le réseau privé
chmod 400 ~/vm-db-key.pem
ssh -i ~/vm-db-key.pem azureuser@10.0.2.4
```

**Résultat : connexion réussie** — confirme que le port 22 TCP depuis `10.0.1.0/24` est bien autorisé par NSG-db-v2, et que le routage privé entre les subnets fonctionne au sein du VNet.

> 📸 `09-ssh-jump-vm-db.png`

---

## Concepts clés démontrés

**Segmentation réseau** — découper un espace d'adressage /16 en subnets dédiés permet d'isoler les tiers applicatifs au niveau réseau (exposition internet vs. interne).

**NSG comme firewall stateful** — les règles sont évaluées par ordre de priorité (valeur la plus basse en premier). Les NSGs suivent l'état des connexions : une connexion TCP autorisée en entrée permet automatiquement le trafic retour sans règle outbound explicite.

**Défense en profondeur** — VM-db est injoignable depuis internet mais pleinement accessible depuis VM-web. Ce schéma reproduit une architecture web/base de données réelle où la couche données n'est jamais exposée publiquement.

**ICMP vs TCP** — les NSGs opèrent au niveau de la couche transport. Autoriser le port 22 TCP n'autorise pas implicitement l'ICMP. Le ping qui échoue pendant que le SSH fonctionne est le comportement attendu sur un subnet verrouillé.

**IPs privées et DHCP** — Azure réserve les quatre premières adresses de chaque subnet (adresse réseau, passerelle, DNS, broadcast). La première IP assignable est toujours `.4` (ex : `10.0.1.4`, `10.0.2.4`).

**Permissions clé SSH sous WSL** — `chmod` ne fonctionne pas sur les fichiers stockés sur les chemins Windows montés (`/mnt/c/`). Les clés doivent être copiées dans le home Linux avant d'appliquer les permissions.

---

## Nettoyage

> ⚠️ Supprimer toutes les ressources immédiatement après documentation pour préserver les crédits Free Trial.

**Portail** → *Resource groups* → `rg-lab-reseau` → **Delete resource group** → confirmer en tapant le nom.

Cette opération supprime tout en une fois : VMs, NICs, IPs publiques, NSGs, VNet, subnets, disques, clés SSH.

Coût estimé pour ce lab (Standard D2s_v3 × 2, ~1 heure) : moins de **0,25$**.

> 📸 `10-cleanup.png`

---

## Structure du repo

```
azure-lab-vnet-nsg/
├── README.md
└── assets/
    ├── 02-vnet-subnets.png
    ├── 03-nsg-web-rules.png
    ├── 04-nsg-db-rules.png
    ├── 05-nsg-associations.png
    ├── 06-vms-running.png
    ├── 07-ssh-vm-web.png
    ├── 08-ping-vm-db.png
    ├── 09-ssh-jump-vm-db.png
    └── 10-cleanup.png
```

---

## Auteur

**Danee Aya Samy** — Étudiant ingénieur informatique (CESI Lyon)  
Stage : Agents IA & Infrastructure Cloud, EM Lyon Business School  
[LinkedIn](https://linkedin.com/in/) · [GitHub](https://github.com/)
