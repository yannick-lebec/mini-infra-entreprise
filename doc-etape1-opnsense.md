# Documentation Technique — Étape 1
## Installation et configuration du Firewall OPNsense

**Projet :** Mini Infrastructure d'Entreprise  
**Auteur :** Yannick Le Bec  
**Date :** Mai 2026  
**Environnement :** VirtualBox sur Windows

---

## 1. Objectif

Mettre en place un firewall/routeur OPNsense dans un environnement virtualisé, segmenter le réseau en zones LAN et DMZ, et valider la connectivité entre les VMs.

---

## 2. Matériel et logiciels utilisés

| Composant | Détail |
|---|---|
| Hyperviseur | Oracle VirtualBox |
| OS hôte | Windows 11 |
| Firewall | OPNsense 26.1.6 (amd64) |
| Client test | Kali Linux 2025.3 / Ubuntu 26.04 LTS |

---

## 3. Architecture réseau

```
Internet (simulé)
      │
   [OPNsense] ← Firewall / Routeur / DHCP
   WAN : 10.0.3.2 (NAT VirtualBox)
      │
   ┌──┴──────────────┐
 [LAN]             [DMZ]
 192.168.1.0/24    192.168.2.0/24
      │
 [Kali / Ubuntu]
 192.168.1.50 / DHCP
```

---

## 4. Configuration de la VM OPNsense

### 4.1 Paramètres de la VM

| Paramètre | Valeur |
|---|---|
| Nom | opnsense |
| Type | BSD / FreeBSD (64-bit) |
| RAM | 1024 Mo |
| Disque | 16 Go (VDI dynamique) |

### 4.2 Configuration des interfaces réseau

| Adaptateur VirtualBox | Mode | Réseau | Interface OPNsense |
|---|---|---|---|
| Adaptateur 1 | Réseau interne | `LAN` | em0 → LAN |
| Adaptateur 2 | NAT | — | em1 → WAN |
| Adaptateur 3 | Réseau interne | `DMZ` | em2 → DMZ |

> ⚠️ **Point critique :** L'Adaptateur 1 doit être en Réseau interne LAN (et non NAT) pour qu'OPNsense assigne correctement em0 au LAN. L'ordre des adaptateurs détermine l'assignation des interfaces em0/em1/em2.

---

## 5. Installation d'OPNsense

### 5.1 Téléchargement de l'ISO

- Source : [opnsense.org/download](https://opnsense.org/download)
- Architecture : **amd64**
- Type d'image : **dvd**
- Mirror : LeaseWeb (NL) ou Frankfurt (DE)
- Fichier obtenu : `OPNsense-26.1.6-dvd-amd64.iso.bz2`
- Extraction : 7-Zip → `OPNsense-26.1.6-dvd-amd64.iso`

### 5.2 Procédure d'installation

1. Démarrer la VM sur l'ISO
2. À l'invite `login:` → taper `installer` / mot de passe : `opnsense`
3. Sélection du clavier → **French**
4. Méthode de partitionnement → **UFS**
5. Disque cible → **ada0** (VBOX HARDDISK 16 Go)
6. Attendre la copie du système (~100%)
7. Définir le mot de passe root
8. Terminer l'installation → reboot
9. Éjecter l'ISO avant le redémarrage

---

## 6. Configuration réseau post-installation

### 6.1 Assignation des interfaces (option 1 du menu)

Depuis le menu console OPNsense :
- LAGGs : **N**
- VLANs : **N**
- Interface WAN : **em1**
- Interface LAN : **em0**

### 6.2 Configuration de l'IP LAN (option 2 du menu)

| Paramètre | Valeur |
|---|---|
| Interface | LAN (em0) |
| DHCP | Non (IP statique) |
| Adresse IPv4 | 192.168.1.1 |
| Masque | /24 (255.255.255.0) |
| Gateway | — (vide pour LAN) |
| IPv6 | Non |
| Serveur DHCP | Activé |
| Plage DHCP | 192.168.1.100 → 192.168.1.200 |
| Protocole GUI | HTTPS (conservé) |

---

## 7. Configuration du client (Kali Linux / Ubuntu)

### 7.1 Configuration réseau Kali

| Adaptateur | Mode | Réseau |
|---|---|---|
| Adaptateur 1 | NAT | Internet |
| Adaptateur 2 | Host-Only | 192.168.56.x |
| Adaptateur 3 | Réseau interne | `LAN` |

### 7.2 Configuration IP statique sur eth2 (Kali)

```bash
# Via NetworkManager
sudo nmcli connection add type ethernet ifname eth2 \
  con-name LAN ip4 192.168.1.50/24 gw4 192.168.1.1
sudo nmcli connection up LAN
```

### 7.3 Ubuntu — configuration automatique

Avec l'Adaptateur 1 en Réseau interne `LAN`, Ubuntu obtient une IP automatiquement via le DHCP d'OPNsense (plage 192.168.1.100-200).

---

## 8. Tests de validation

### 8.1 Test de connectivité

```bash
# Depuis Kali/Ubuntu vers OPNsense
ping 192.168.1.1

# Résultat attendu
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=2.61 ms
8 packets transmitted, 8 received, 0% packet loss
```

### 8.2 Accès à l'interface web

- URL : `https://192.168.1.1`
- Login : `root`
- Mot de passe : défini lors de l'installation
- Accepter l'avertissement certificat SSL auto-signé

### 8.3 Vérification du dashboard

✓ Interface LAN active → 192.168.1.1/24  
✓ Interface WAN active → DHCP (10.0.3.2)  
✓ Serveur DHCP fonctionnel  
✓ Firewall opérationnel  
✓ Services actifs : DNS/DHCP, SSH, Packet Filter  

---

## 9. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Mot de passe refusé au login | Clavier QWERTY au lieu d'AZERTY | Sélectionner le clavier French lors de l'installation |
| OPNsense ne détecte qu'une interface | Adaptateurs LAN/DMZ non ajoutés à la VM | Ajouter les adaptateurs 2 et 3 dans la config VirtualBox |
| Ping "Destination Host Unreachable" | LAN d'OPNsense assigné sur l'Adaptateur 1 (NAT) | Inverser Adaptateur 1 (LAN) et Adaptateur 2 (NAT) |
| IP non persistante sur Kali | NetworkManager remplace le système networking | Utiliser `nmcli` pour créer la connexion |
| Interface web inaccessible depuis le PC hôte | PC hôte non connecté au réseau interne LAN | Accéder depuis une VM cliente sur le réseau LAN |

---

## 10. Captures d'écran

| Capture | Description |
|---|---|
| `captures/dashboard-opnsense.png` | Dashboard OPNsense — interfaces LAN/WAN actives, services en ligne |
| `captures/ping-kali-opnsense.png` | Ping 192.168.1.1 depuis Kali — 6 paquets, 0% perte |
| `captures/vbox-adapter1-lan.png` | VirtualBox — Adaptateur 1 : Réseau interne LAN |
| `captures/vbox-adapter2-nat.png` | VirtualBox — Adaptateur 2 : NAT (WAN) |
| `captures/vbox-adapter3-dmz.png` | VirtualBox — Adaptateur 3 : Réseau interne DMZ |

> ⚠️ **Point d'attention relevé sur le dashboard :** OPNsense affiche le bandeau *"You are currently running in live media mode. A reboot will reset the configuration."*  
> Cela indique que la VM tourne depuis l'ISO sans que la configuration soit persistée sur le disque. **Si la VM redémarre, toute la configuration est perdue.**  
> **Action requise :** Vérifier que l'installation sur le disque `ada0` a bien été effectuée et que la VM démarre sur le disque, pas sur l'ISO. Retirer l'ISO du lecteur optique dans les paramètres VirtualBox.

---

## 11. Prochaine étape

**Semaine 2 — Active Directory**  
Installation et configuration de Windows Server 2019 comme contrôleur de domaine (AD DS), configuration DNS interne, création d'utilisateurs et de GPO.

---

*Documentation rédigée dans le cadre d'un projet portfolio — Alternance Technicien Systèmes & Réseaux*
