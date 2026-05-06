# Audit de Sécurité — Mini Infrastructure d'Entreprise

> Yannick Le Bec | Ada Tech School | Projet Portfolio TSR

---

## Résumé exécutif

L'architecture proposée est **solide pour un projet portfolio** et suit les bonnes pratiques fondamentales (segmentation LAN/DMZ, firewall central). Cependant, plusieurs vecteurs d'attaque typiques en entreprise méritent d'être adressés, à la fois pour renforcer la sécurité réelle et pour démontrer une maturité technique aux recruteurs.

**Score global : 6/10** — bonne base, lacunes comblables.

---

## 1. Architecture réseau

### Points positifs
- La segmentation LAN / DMZ est le bon réflexe fondamental.
- pfSense en frontal centralise le contrôle du flux.
- Le serveur web est isolé du LAN — il ne peut pas accéder directement à l'AD.

### Risques identifiés

**Pas de zone de gestion (OOB / Management VLAN)**

Le pfSense lui-même est administré depuis le LAN, ce qui signifie qu'un poste compromis sur le LAN peut atteindre l'interface d'administration du firewall. En entreprise, on isole l'administration dans un VLAN dédié (souvent `192.168.100.0/24`).

**Recommandation :** Créer un 3e réseau `MGMT` dans VirtualBox et n'autoriser l'accès à l'interface web de pfSense que depuis ce VLAN.

**Flux DMZ → LAN non explicitement bloqué**

Si une règle "permettre tout" est créée sur l'interface DMZ de pfSense par défaut (ce que fait pfSense lors du wizard), le serveur web compromis peut initier des connexions vers l'AD.

**Recommandation :** Règle explicite `DENY ALL` sur l'interface DMZ vers le LAN, avec uniquement les exceptions nécessaires (ex. : DNS).

---

## 2. pfSense — Firewall

### Risques identifiés

**Interface d'administration exposée sur le LAN sans HTTPS forcé / 2FA**

Par défaut, pfSense écoute sur le port 443 mais peut accepter HTTP sur 80. Un accès non chiffré sur un réseau interne reste un risque (ARP spoofing, MITM).

**Recommandation :**
- Forcer HTTPS uniquement
- Changer le port d'admin (ex. : 8443)
- Activer l'authentification à deux facteurs (pfSense supporte TOTP via le package `FreeRadius`)
- Limiter l'accès admin aux IPs de la zone MGMT uniquement

**Règles firewall trop permissives par défaut**

Le wizard pfSense crée une règle `LAN → any → ALLOW` qui autorise tout le trafic sortant depuis le LAN. En entreprise, on applique le principe du moindre privilège.

**Recommandation :** Passer à une politique par défaut `DENY` et whitelister uniquement les flux nécessaires :
```
LAN → WAN : HTTP, HTTPS, DNS autorisés
LAN → DMZ : DENY (sauf cas spécifiques)
DMZ → WAN : HTTP, HTTPS autorisés (màj paquets)
DMZ → LAN : DENY ALL
```

**Pas de détection d'intrusion (IDS/IPS)**

pfSense intègre Suricata et Snort gratuitement. Sans IDS, une attaque traversant le firewall est invisible.

**Recommandation :** Installer le package **Suricata** sur pfSense, activer les règles ET Open sur les interfaces WAN et DMZ.

**Pas de logs centralisés**

Les logs pfSense sont stockés en RAM (perdus au redémarrage).

**Recommandation :** Configurer l'export Syslog vers un serveur de logs (même une instance Syslog sur le Windows Server fera l'affaire pour le projet).

---

## 3. Active Directory (Windows Server 2022)

C'est le composant le plus critique — si l'AD est compromis, l'infrastructure entière l'est.

### Risques identifiés

**Compte Administrateur de domaine utilisé au quotidien**

Le réflexe classique des débutants est de tout faire avec le compte `Administrator` du domaine. C'est une cible directe pour les attaques Pass-the-Hash et les mouvements latéraux.

**Recommandation :**
- Créer un compte admin dédié pour les tâches AD (ex. `adm-yannick`)
- Désactiver le compte `Administrator` local sur les postes clients via GPO
- Ne jamais se connecter avec un compte DA sur un poste client

**Pas de tiering du modèle d'administration**

Le modèle Microsoft recommande 3 niveaux :
- **Tier 0** : Contrôleurs de domaine, AD (comptes SA/DA)
- **Tier 1** : Serveurs (Linux, web)
- **Tier 2** : Postes utilisateurs

**Recommandation :** Appliquer ce modèle même simplifié — les admins Tier 0 ne se connectent que sur les DCs, jamais sur le client Windows 10.

**Politique de mots de passe insuffisante (défaut AD)**

Par défaut, l'AD autorise des mots de passe de 7 caractères minimum sans complexité maximale.

**Recommandation GPO :** (`Computer Configuration > Windows Settings > Security Settings > Account Policies`)
```
Longueur minimale : 14 caractères
Complexité : Activée
Durée de vie max : 90 jours
Historique : 12 mots de passe
Verrouillage après : 5 tentatives
```

**SMBv1 potentiellement actif**

Windows Server 2022 a SMBv1 désactivé par défaut, mais une mauvaise configuration peut le réactiver. SMBv1 est vulnérable à EternalBlue (WannaCry).

**Recommandation :** Vérifier et forcer la désactivation de SMBv1 :
```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false
```

**Absence de Microsoft LAPS**

Sans LAPS, tous les postes clients ont le même mot de passe administrateur local — une seule compromission = mouvement latéral sur tous les postes.

**Recommandation :** Déployer **Microsoft LAPS** (natif dans Windows Server 2022) pour randomiser les mots de passe admins locaux.

**Kerberoasting possible**

Si des comptes de service ont des SPN configurés avec des mots de passe faibles, un attaquant peut demander des tickets de service et les cracker hors ligne.

**Recommandation :** Utiliser des **gMSA (group Managed Service Accounts)** pour les comptes de service — le mot de passe est géré automatiquement par l'AD (120 caractères aléatoires).

---

## 4. Serveur Web en DMZ (Debian/Nginx)

### Risques identifiés

**Nginx expose la version par défaut**

Par défaut, Nginx inclut sa version dans les headers HTTP (`Server: nginx/1.24.0`), ce qui facilite le ciblage CVE.

**Recommandation :** Dans `/etc/nginx/nginx.conf` :
```nginx
server_tokens off;
```

**Pas de TLS / HTTPS**

Un site en HTTP expose les données en clair et est vulnérable aux attaques MITM. Même pour un lab, configurer TLS démontre une compétence attendue.

**Recommandation :** Générer un certificat auto-signé pour le lab :
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt
```

**Utilisateur `www-data` avec trop de permissions**

Vérifier que Nginx tourne en tant que `www-data` (et non root) et que ce compte n'a pas de shell interactif ni de sudo.

**Recommandation :**
```bash
# Vérifier le processus
ps aux | grep nginx
# www-data ne doit pas avoir de shell
grep www-data /etc/passwd  # doit afficher /usr/sbin/nologin
```

**Mises à jour automatiques non configurées**

Un serveur en DMZ exposé doit être patché en priorité.

**Recommandation :**
```bash
apt install unattended-upgrades
dpkg-reconfigure unattended-upgrades
```

**Accès SSH depuis la DMZ non restreint**

Par défaut, SSH écoute sur toutes les interfaces. Une DMZ compromise peut servir de rebond si SSH est accessible depuis le WAN.

**Recommandation :**
- Désactiver l'authentification par mot de passe (`PasswordAuthentication no`)
- N'autoriser SSH que depuis le LAN (via règle pfSense)
- Changer le port SSH (réduit le bruit dans les logs)

---

## 5. Poste client Windows 10

### Risques identifiés

**Windows Defender peut être désactivé par GPO involontairement**

Certaines GPO "d'optimisation" désactivent Defender. Vérifier qu'il reste actif.

**Recommandation GPO :** Forcer l'activation de Defender et des mises à jour automatiques via `Computer Configuration > Administrative Templates > Windows Components > Windows Defender Antivirus`.

**Pas de chiffrement disque**

Si le poste est compromis physiquement, les données sont accessibles.

**Recommandation :** Activer BitLocker via GPO (fonctionnel même en VM pour démontrer la compétence) avec stockage de la clé de récupération dans l'AD.

---

## 6. Points transversaux

### Absence de monitoring centralisé

Sans SIEM ou collecte de logs, aucune visibilité sur ce qui se passe dans l'infra.

**Recommandation :** Installer **Wazuh** (SIEM open source, gratuit) sur une VM légère — excellent ajout portfolio, collecte les logs AD, pfSense et Linux dans une interface unifiée.

### Pas de sauvegarde testée

**Recommandation :** Documenter une procédure de sauvegarde/restauration de l'AD (snapshot VirtualBox + `ntdsutil` ou Windows Server Backup).

---

## Récapitulatif des actions prioritaires

| Priorité | Action | Composant |
|---|---|---|
| 🔴 Critique | Règle `DENY` DMZ → LAN explicite | pfSense |
| 🔴 Critique | Ne jamais utiliser le compte DA au quotidien | AD |
| 🔴 Critique | Désactiver authentification SSH par mot de passe | Debian |
| 🟠 Élevé | Installer Suricata sur pfSense | pfSense |
| 🟠 Élevé | Déployer Microsoft LAPS | AD |
| 🟠 Élevé | GPO : politique de mots de passe forte | AD |
| 🟡 Moyen | Configurer TLS sur Nginx | Debian |
| 🟡 Moyen | `server_tokens off` dans Nginx | Debian |
| 🟡 Moyen | Syslog centralisé | pfSense + Linux |
| 🟢 Bonus portfolio | Installer Wazuh SIEM | VM dédiée |
| 🟢 Bonus portfolio | Configurer BitLocker + stockage clé dans AD | Client |

---

## Conclusion

L'architecture de base est correcte et montre que tu maîtrises les concepts fondamentaux (segmentation, firewall, AD). En ajoutant les corrections identifiées — particulièrement la règle DMZ→LAN, LAPS, et Suricata — tu passeras d'une infra "ça fonctionne" à une infra "sécurisée par conception", ce qui est exactement ce qu'un jury ou un recruteur TSR veut voir.
