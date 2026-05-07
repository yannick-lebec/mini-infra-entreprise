# Documentation Technique — Étape 2
## Installation et configuration de l'Active Directory (Windows Server 2022)

**Projet :** Mini Infrastructure d'Entreprise  
**Auteur :** Yannick Le Bec  
**Date :** Mai 2026  
**Environnement :** VirtualBox sur Windows

---

## 1. Objectif

Promouvoir Windows Server 2022 en contrôleur de domaine Active Directory, configurer le DNS interne, créer des utilisateurs et des groupes de sécurité pour simuler une infrastructure d'entreprise réelle.

---

## 2. Matériel et logiciels utilisés

| Composant | Détail |
|---|---|
| Hyperviseur | Oracle VirtualBox |
| OS | Windows Server 2022 Standard Evaluation |
| Mode | Core (sans interface graphique) |
| Rôle installé | AD DS + DNS |
| Domaine | lab.local |

---

## 3. Architecture réseau

```
[OPNsense Firewall]
  192.168.1.1 (LAN)
       │
  [LAN 192.168.1.0/24]
       │
[Windows Server 2022]
  192.168.1.10 (IP fixe)
  Contrôleur de domaine lab.local
  Serveur DNS interne
```

---

## 4. Configuration de la VM Windows Server 2022

### 4.1 Paramètres de la VM

| Paramètre | Valeur |
|---|---|
| Nom | WindowsServer2022 |
| OS | Windows Server 2022 (64-bit) |
| RAM | 2048 Mo minimum |
| Disque | Existant |

### 4.2 Configuration réseau

| Adaptateur | Mode | Réseau |
|---|---|---|
| Adaptateur 1 | Réseau interne | `LAN` |
| Adaptateur 2 | NAT | Internet |

---

## 5. Configuration IP statique

Depuis le menu **SConfig → option 8 (Paramètres réseau) → Carte 1** :

| Paramètre | Valeur |
|---|---|
| Type | Statique |
| Adresse IP | 192.168.1.10 |
| Masque | 255.255.255.0 |
| Passerelle | 192.168.1.1 |
| DNS préféré | 192.168.1.1 |
| DNS auxiliaire | 8.8.8.8 |

### Validation de la connectivité

```powershell
ping 192.168.1.1
# Résultat : 4 paquets envoyés, 4 reçus, 0% de perte
```

---

## 6. Installation du rôle AD DS

Depuis PowerShell (SConfig → option 15) :

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

**Résultat :**
```
Success : True
Restart Needed : No
Exit Code : Success
Feature Result : Services AD DS, Gestion de stratégie de groupe
```

---

## 7. Promotion en contrôleur de domaine

```powershell
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns:$true `
  -Force:$true
```

**Paramètres du domaine :**

| Paramètre | Valeur |
|---|---|
| Nom de domaine FQDN | lab.local |
| Nom NetBIOS | LAB |
| Niveau fonctionnel | Windows Server 2016 (par défaut) |
| DNS intégré | Oui |
| Mot de passe DSRM | Défini lors de l'installation |

Le serveur redémarre automatiquement après la promotion.

**Vérification après redémarrage :**

Menu SConfig → option 1 affiche :
```
Domaine : lab.local ✓
```

---

## 8. Création des utilisateurs AD

> ⚠️ **Note sécurité :** Les mots de passe ci-dessous sont des exemples de lab. Ne jamais utiliser des mots de passe en clair dans une documentation publiée sur un dépôt public.

### 8.1 Utilisateur 1

```powershell
New-ADUser `
  -Name "jdupont" `
  -AccountPassword (ConvertTo-SecureString "<mot-de-passe>" -AsPlainText -Force) `
  -Enabled $true
```

### 8.2 Utilisateur 2

```powershell
New-ADUser `
  -Name "mmartin" `
  -AccountPassword (ConvertTo-SecureString "<mot-de-passe>" -AsPlainText -Force) `
  -Enabled $true
```

### 8.3 Vérification des utilisateurs

```powershell
Get-ADUser -Filter *
```

**Résultat :**
```
Name         : jdupont
Enabled      : True
DistinguishedName : CN=jdupont,CN=Users,DC=lab,DC=local

Name         : mmartin
Enabled      : True
DistinguishedName : CN=mmartin,CN=Users,DC=lab,DC=local
```

---

## 9. Création du groupe de sécurité

### 9.1 Création du groupe

```powershell
New-ADGroup -Name "Employes" -GroupScope Global -GroupCategory Security
```

### 9.2 Ajout des membres

```powershell
Add-ADGroupMember -Identity "Employes" -Members jdupont,mmartin
```

### 9.3 Vérification du groupe

```powershell
Get-ADGroupMember -Identity "Employes"
```

**Résultat :**
```
name           : jdupont
objectClass    : user
SamAccountName : jdupont

name           : mmartin
objectClass    : user
SamAccountName : mmartin
```

---

## 10. Structure AD finale

```
lab.local (Domaine)
└── CN=Users
    ├── jdupont (Utilisateur) ✓
    ├── mmartin (Utilisateur) ✓
    └── Employes (Groupe de sécurité Global) ✓
        ├── jdupont
        └── mmartin
```

---

## 11. Tests de validation

| Test | Commande | Résultat |
|---|---|---|
| Connectivité OPNsense | `ping 192.168.1.1` | ✓ 0% perte |
| Liste utilisateurs AD | `Get-ADUser -Filter *` | ✓ jdupont, mmartin |
| Membres du groupe | `Get-ADGroupMember -Identity "Employes"` | ✓ 2 membres |
| Domaine actif | SConfig → option 1 | ✓ lab.local |

---

## 12. Captures d'écran

| Capture | Description |
|---|---|
| `captures/ad-get-aduser.png` | `Get-ADUser -Filter *` — utilisateurs jdupont et mmartin actifs (`Enabled: True`) |
| `captures/ad-groupe-employes.png` | `Get-ADGroupMember "Employes"` — 2 membres confirmés |
| `captures/sconfig-domaine-lab.png` | SConfig — Domaine : lab.local actif |
| `captures/ping-winserver-opnsense.png` | Ping 192.168.1.1 — 4 paquets, 0% perte |

---

## 13. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Mot de passe oublié (ancienne VM) | VM précédente avec domaine RUE25 | Utilisation d'une nouvelle VM Windows Server 2022 |
| Commandes longues à taper | Mode Core sans interface graphique | Utilisation des commandes PowerShell simplifiées |

---

## 14. Prochaine étape

**Semaine 3 — Serveur Web en DMZ**  
Installation et configuration d'un serveur web Nginx sur Debian/Ubuntu dans la zone DMZ (192.168.2.0/24), configuration des règles firewall OPNsense pour autoriser l'accès HTTP/HTTPS depuis le LAN vers la DMZ.

---

*Documentation rédigée dans le cadre d'un projet portfolio — Alternance Technicien Systèmes & Réseaux*
