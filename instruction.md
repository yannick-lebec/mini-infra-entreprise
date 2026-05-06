# Projet Portfolio — Mini Infrastructure d'Entreprise
> Yannick Le Bec | Ada Tech School | Alternance Technicien Systèmes & Réseaux

---

## 🎯 Objectif

Simuler une infrastructure réseau d'entreprise complète dans VirtualBox, documenter chaque étape, et présenter le tout dans un portfolio en ligne. Ce projet vise à démontrer des compétences concrètes en administration systèmes et réseaux pour décrocher une alternance Technicien S&R en Île-de-France.

---

## 🖥️ Environnement de travail

- **OS hôte :** Windows
- **Hyperviseur :** VirtualBox
- **Niveau réseau :** Intermédiaire (VLAN, DHCP, routage)

---

## 🏗️ Architecture cible

```
Internet (simulé)
      │
   [pfSense] ← Firewall / Routeur / DHCP
      │
   [Switch virtuel VirtualBox]
   ┌──┴──────────────┐
 [LAN]             [DMZ]
   │                 │
[Windows Server   [Serveur Web
  2022 — AD/DNS]   Debian/Nginx]
   │
[Client Windows 10
 — joint au domaine]
```

### VMs à créer

| VM | Rôle | OS | RAM conseillée |
|---|---|---|---|
| pfSense | Firewall, routeur, DHCP | pfSense CE | 1 Go |
| Windows Server | Active Directory, DNS, GPO | Windows Server 2022 | 2 Go |
| Serveur Web | Site web en DMZ | Debian 12 / Ubuntu Server | 1 Go |
| Client | Poste utilisateur | Windows 10 | 2 Go |

### Plan d'adressage réseau

| Réseau | Plage IP | Rôle |
|---|---|---|
| WAN | 10.0.2.0/24 | Accès Internet simulé (NAT VirtualBox) |
| LAN | 192.168.1.0/24 | Réseau interne entreprise |
| DMZ | 192.168.2.0/24 | Zone démilitarisée (serveurs exposés) |

---

## 📋 Compétences démontrées

- ✅ Configuration firewall et règles réseau (pfSense)
- ✅ Segmentation réseau LAN / DMZ
- ✅ Active Directory, gestion utilisateurs, GPO (Windows Server)
- ✅ Administration serveur Linux (Debian/Nginx)
- ✅ Intégration poste client dans un domaine AD
- ✅ Documentation technique professionnelle

---

## 📅 Planning (3-4 semaines)

### Semaine 1 — pfSense & réseau de base
- [ ] Installer pfSense dans VirtualBox
- [ ] Configurer les interfaces WAN / LAN / DMZ
- [ ] Activer et configurer le serveur DHCP (LAN)
- [ ] Créer les règles firewall de base
- [ ] Tester la connectivité entre les réseaux

### Semaine 2 — Windows Server & Active Directory
- [ ] Installer Windows Server 2022
- [ ] Promouvoir en contrôleur de domaine (AD DS)
- [ ] Configurer le DNS interne
- [ ] Créer des utilisateurs et groupes AD
- [ ] Appliquer des GPO (politiques de groupe)

### Semaine 3 — Serveur Web en DMZ
- [ ] Installer Debian 12 / Ubuntu Server
- [ ] Installer et configurer Nginx
- [ ] Déployer une page web de démonstration
- [ ] Configurer les règles firewall pfSense pour la DMZ
- [ ] Tester l'isolation DMZ ↔ LAN

### Semaine 4 — Client, tests & documentation
- [ ] Installer Windows 10 client
- [ ] Joindre le poste au domaine AD
- [ ] Vérifier l'application des GPO
- [ ] Tests de sécurité (ping, accès cross-zone)
- [ ] Rédiger la documentation finale
- [ ] Mettre en ligne sur GitHub + portfolio

---

## 📁 Livrables portfolio

- [ ] **README GitHub** avec schéma d'architecture et description
- [ ] **Captures d'écran** commentées de chaque étape
- [ ] **Document de configuration** technique (style rapport technicien)
- [ ] **Site web portfolio** présentant le projet
- [ ] **Vidéo démo** (2-3 minutes, optionnel mais recommandé)

---

## 🌐 Site Portfolio — Structure suggérée

### Pages / Sections

1. **Accueil** — Présentation du projet, objectif, stack technique
2. **Architecture** — Schéma interactif de l'infra, plan d'adressage
3. **Étapes** — Timeline des 4 semaines avec captures et explications
4. **Compétences** — Ce que le projet démontre techniquement
5. **Documentation** — Lien vers GitHub, fichiers de config, rapport

### Stack suggérée pour le site
- **React + TypeScript** (cohérent avec ton stack Ada Tech)
- **Tailwind CSS** pour le style
- **Déployé sur Vercel ou GitHub Pages**

---

## 📝 Notes & Ressources

- ISO pfSense CE : https://www.pfsense.org/download/
- ISO Windows Server 2022 (évaluation 180j) : https://www.microsoft.com/en-us/evalcenter/
- ISO Debian 12 : https://www.debian.org/distrib/
- ISO Windows 10 : https://www.microsoft.com/fr-fr/software-download/windows10
- VirtualBox : https://www.virtualbox.org/

---

## 🚀 Étape en cours

**→ Semaine 1 : Installation et configuration de pfSense**