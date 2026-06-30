<div align="center">

# Lab CCNA - Réseau d'entreprise multi-sites redondant

**Simulation complète d'une infrastructure réseau d'entreprise**  
avec haute disponibilité, segmentation VLAN et sécurisation des accès

![Outil](https://img.shields.io/badge/Outil-Cisco%20Packet%20Tracer-1BA0D7?style=flat-square&logo=cisco&logoColor=white)
![Niveau](https://img.shields.io/badge/Niveau-CCNA1%2FCCNA2-orange?style=flat-square)
![Statut](https://img.shields.io/badge/Statut-En%20cours-yellow?style=flat-square)
![Équipements](https://img.shields.io/badge/Équipements-9-green?style=flat-square)

</div>

---

## Sommaire

- [Présentation](#-présentation)
- [Topologie](#-topologie)
- [Plan d'adressage](#-plan-dadressage)
- [Fonctionnalités mises en œuvre](#-fonctionnalités-mises-en-œuvre)
- [Détail des équipements](#-détail-des-équipements)
- [Sécurité réseau](#-sécurité-réseau)
- [Concepts CCNA couverts](#-concepts-ccna-couverts)
- [Structure du dépôt](#-structure-du-dépôt)
- [Limites connues](#-limites-connues)

---

## Présentation

Ce projet simule l'infrastructure réseau d'une entreprise multi-départements sous **Cisco Packet Tracer**. Il couvre l'essentiel des compétences CCNA1 et CCNA2 : routage inter-VLAN, redondance de passerelle (HSRP), agrégation de liens (EtherChannel), sécurité couche 2 et 3, et administration distante sécurisée via SSH.

### Objectifs
- Concevoir un plan d'adressage IP structuré et cohérent
- Assurer la **haute disponibilité** au niveau du cœur réseau (aucun SPOF)
- Isoler logiquement les flux par département via des **VLANs dédiés**
- Sécuriser les accès physiques et logiques contre les attaques courantes de couche 2

---

## Topologie

```
                        [Internet]
                            |
                         [ISP]
                      203.0.113.1/30
                            |
                       [R1-Edge]
                  G0/1              G0/2
          .131.1/30                    .132.1/30
                |                          |
           [MLS1]  ══════ Po1 LACP ══════ [MLS2]
         HSRP Actif                    HSRP Standby
        .131.2/30                        .132.2/30
          |      |                       |       |
        [S1]   [S2]                   [S3]    [S4]
       VLAN10  VLAN20               VLAN40  VLAN30
         |       |                    |       |
      PC-RH  PC-Compta         PC-Direction  AP WLAN
                                  Srv Local   Laptop
```

> *(Capture de la topologie Packet Tracer à insérer ici)*

---

## Plan d'adressage

### Liens WAN - point-à-point (/30)

| Segment | Réseau | Côté A | Côté B |
|---|---|---|---|
| ISP ↔ R1-Edge | `203.0.113.0/30` | ISP G0/0 `.1` | R1 G0/0 `.2` |
| R1-Edge ↔ MLS1 | `209.237.131.0/30` | R1 G0/1 `.1` | MLS1 G0/1 `.2` (routed) |
| R1-Edge ↔ MLS2 | `209.237.132.0/30` | R1 G0/2 `.1` | MLS2 G0/2 `.2` (routed) |
| ISP ↔ Serveur internet | `8.8.8.0/30` | ISP G0/1 `.1` | Srv `.2` |

### VLANs internes

| VLAN | Nom | Réseau | Passerelle HSRP | Mgmt switch |
|---|---|---|---|---|
| 10 | RH | `192.168.10.0/24` | `.10.254` | S1 → `.10.9` |
| 20 | Compta | `192.168.20.0/24` | `.20.254` | S2 → `.20.9` |
| 30 | Invités | `192.168.30.0/24` | `.30.254` | S4 → `.30.9` |
| 40 | Direction | `192.168.40.0/24` | `.40.254` | S3 → `.40.9` |
| 237 | Natif (trunk) | - | - | - |
| 999 | Poubelle (ports inutilisés) | - | - | - |

### DHCP - plages distribuées par VLAN

| VLAN | Exclusions | Plage distribuée |
|---|---|---|
| 10 | `.1` → `.10` et `.254` | `.11` → `.253` |
| 20 | `.1` → `.10` et `.254` | `.11` → `.253` |
| 30 | `.1` → `.10` et `.254` | `.11` → `.253` |
| 40 | `.1` → `.10`, `.100` et `.254` | `.11` → `.253` (sauf `.100`) |

---

## Fonctionnalités mises en œuvre

### Haute disponibilité

| Technologie | Configuration |
|---|---|
| **HSRPv2** | MLS1 actif sur VLAN 10/20 (prio 150), MLS2 actif sur VLAN 30/40 (prio 150) → load balancing |
| **EtherChannel LACP** | Port-channel 1 (Fa0/23-24) entre MLS1 et MLS2 |
| **Spanning-Tree** | MLS1 = root bridge primaire sur tous les VLANs actifs |
| **Routes statiques redondantes** | AD 1 (chemin principal) / AD 5 (chemin de secours) sur R1-Edge, MLS1 et MLS2 |

### Services réseau

| Service | Détail |
|---|---|
| **DHCP** | Pool dédié par VLAN, passerelle = IP virtuelle HSRP |
| **DNS** | Pointage vers `8.8.8.8` (serveur internet simulé) |
| **Routage inter-VLAN** | Via SVI sur MLS1/MLS2 avec `ip routing` activé |

---

## Sécurité réseau

### Couche 2 - switches d'accès (S1, S2, S3)

```
Port-security    : max 1 adresse MAC par port (sticky + violation shutdown)
DHCP Snooping    : actif sur le VLAN du switch, port trunk marqué trusted
DAI              : Dynamic ARP Inspection, anti-ARP spoofing
CDP/LLDP         : désactivés sur les ports orientés utilisateurs
VLAN 999         : ports inutilisés isolés dans un VLAN poubelle
```

### Couche 3 - isolation VLAN Invités (S4)

```cisco
ip access-list extended BLOCK_INVITES
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 permit ip any any
```
> Appliquée en `in` sur l'interface VLAN 30 : les invités accèdent à Internet mais sont bloqués vers les VLANs internes.

### Administration distante

```
SSHv2            : transport input ssh sur toutes les lignes VTY
AAA local        : username admin + enable secret (chiffré)
Bannière MOTD    : avertissement légal sur tous les équipements
Min. password    : security password min-length 10
no ip domain-lookup : désactivé pour éviter les délais DNS en CLI
```

---

## Détail des équipements

| Équipement | Modèle | Rôle | Points clés |
|---|---|---|---|
| **ISP** | Cisco 2911 | Fournisseur d'accès simulé | Route statique vers `192.168.0.0/16` via R1 |
| **R1-Edge** | Cisco 2911 | Routeur de bordure | Double liaison vers MLS1/MLS2, SSHv2 |
| **MLS1** | Cisco 3560 (L3) | Cœur réseau actif | HSRP actif V10/V20, root STP, DHCP, Po1 |
| **MLS2** | Cisco 3560 (L3) | Cœur réseau standby | HSRP actif V30/V40, DHCP, Po1 |
| **S1** | Cisco 2960 | Accès VLAN 10 - RH | Port-security, DHCP snooping, DAI |
| **S2** | Cisco 2960 | Accès VLAN 20 - Compta | Port-security, DHCP snooping, DAI |
| **S3** | Cisco 2960 | Accès VLAN 40 - Direction | Port-security, DHCP snooping, DAI |
| **S4** | Cisco 2960 | Accès VLAN 30 - Invités | ACL isolation inter-VLAN, AP WLAN WPA2 |

---

## Concepts CCNA couverts

| Domaine | Concept |
|---|---|
| **Switching L2** | VLANs, trunking 802.1Q, VLAN natif, port-security, DHCP snooping, DAI |
| **Switching L3** | Routage inter-VLAN via SVI, `ip routing`, interfaces routées (`no switchport`) |
| **Redondance** | HSRPv2, EtherChannel LACP, Spanning-Tree root bridge |
| **Routage** | Routes statiques, distance administrative, routes par défaut |
| **WAN** | Liens point-à-point /30, simulation ISP |
| **Sécurité** | ACL étendues, SSHv2, AAA local, VLAN poubelle, CDP/LLDP désactivés |
| **Services** | DHCP (pools par VLAN, exclusions), DNS |
| **Administration** | `ip default-gateway` sur switches L2, bannières MOTD, `no ip domain-lookup` |

---

## Structure du dépôt

```
lab-ccna-reseau-entreprise/
├── README.md
├── configs/
│   ├── Config_ISP.txt
│   ├── Config_R1_Edge.txt
│   ├── Config_MLS1.txt
│   ├── Config_MLS2.txt
│   ├── Config_S1.txt
│   ├── Config_S2.txt
│   ├── Config_S3.txt
│   └── Config_S4.txt
└── captures/
    ├── topologie.png
    ├── show-ip-interface-brief.png
    ├── show-standby-brief.png
    ├── show-etherchannel-summary.png
    ├── show-vlan-brief.png
    ├── show-spanning-tree.png
    ├── show-port-security.png
    ├── show-ip-dhcp-binding.png
    ├── show-access-lists.png
    ├── ping-internet-ok.png
    ├── ping-invites-bloque.png
    └── ssh-connexion.png
```

---

## Limites connues

- [ ] **ISP** - aucune configuration SSH (pas de clé RSA générée)
- [ ] **MLS2** - vérifier la génération de la clé RSA (risque identique à R1-Edge)
- [ ] **S1, S2, S3** - faute de frappe `switchport acces vlan 999` à corriger en `switchport access vlan 999`
- [ ] Ajouter les captures de validation dans `captures/`

---

<div align="center">

*Projet réalisé dans le cadre d'un apprentissage CCNA1/CCNA2 - Cisco Packet Tracer*

</div>
