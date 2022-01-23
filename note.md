## Qui controle AD ?

1 - Compte à privilege sur AD\
2 - Controleur de domaine (physique)\
3 - Dependance de sécurité (SCCM)\
4 - Permission sur objet (permet privilege)

## Principe de tier

Tier 0 : Système critique (serveur AD, serveur certificat, ADFS, système avec dépendance sécurité, poste administration de système critique)\
Tier 1 : Serveur fichier et autres serveurs (propres compte pour l'administration)\
Tier 2 : Station de travail

## Chronolgie de l'attaque

1 - Préparation et reco\
2 - Première ressource compromise\
3 - Compte administrateur compromis (24h à 48h)\
4 - Détection de la compromission (200 jours)

**But du jeu :**  Rendre la partie 3 longue et difficile, rendre non rentable l'attaque\
Le détecter le plus vite possible

## Contrer la reconnaissance

Voir architecture de l'AD : bloodhound (https://github.com/BloodHoundAD/BloodHound)
=> Affiche les chemins permettant d'arriver aux comptes domains admin

Enumération des comptes et système permet à l'attaquant d'élaborer un plan d'attaque

---

LDAP expose l'annuaire et permet l'énumération de compte :\
1. Port 389 et 3268 (catalogue global)
2. 636 et 3269 (version TLS)
3. Enumération possible qu'après authentification Utilisateur
4. Enumération anonyme désativé depuis server 2003
	1. Sauf si dSHeuristics (7eme caractère à 1)
5. Infos dispo sans auth (RootDSE) non désactivale
6. SAM-R (security account management remote)
	1. Interface permettant énumération de compte
	2. Vérifier membres du groupe "Pre-windows 2000 Compatible Access)
	3. Si il contient Principal de sécurité "Anounymous Logon" = énumération sans auth possible
	4. Seul "Authenticated User" doit être membre du groupe (ou autre bien identifié)
7. Via DNS
	1. Demande de résolution anonymes en testant des noms commun
	2. Transfert de zone DNS
		1. A controler via propriété de la zone DNS / transfert de zone
		2. Si transfert authoriser : regarder journal d'évenement DNS (observateur evenement / event 6001)

---

Enumération SMB :<br>
1. Permet de voir les utilisateurs/IP qui sont connecté à des partage SMB
2. Bloodhound se sert de cette liste pour voir les "has Session" et identifier chemin d'attaque
3. Exemple : <code>nmap -p 445 --script-enum-sessions.nse --script-args smbuser=normaluser,smbpass=Pa$$w0rd IP-DC</code>
4. Pourquoi autant d'users sur le DC en partage ? Car un des partages du DC c'est "sysvol", utile pour DL paramètre de stratégie de groupe (quand PC/User se connecte, il se connecte à SYSVOL)
5. Désactiver cette fonctionalité via powershell (on retire la permission d'énumération SMB aux users auth) :
<code>Get-Module -Name NetCease | Format-List</code><br>
<code>Get-NetSessionEnumPermission | Out-GridView</code><br>
<code>Set-NetSessionEnumPemission</code><br>
Necessite reboot du serveur<br>
A inclure en stratégie globale, sur l'ensemble des serveurs 445 SMB

---

Enumération SAM-R :<br>
1. Outil : SuperScan 4 par exemple
2. Vu précédement pour anonyme, à désactiver via groupe built-in "Pre windows 2000 Compatible Access)
3. Restreindre l'API SAM à des groupes particuliers :
	1. Via GPO : Accès réseau : restreindre les clients autorisés à effectuer des appels distants vers SAM
	2. Ne pas appliquer sur les DC !

---

Recommandations :
1. Ne pas mutualiser les DC avec d'autres services
	1. Sauf service DNS
	2. DANGER : augmente surface d'attaque
	3. DANGER : plus de mise à jour avec reboot / impact dispo serveur
	4. DANGER : Pas de granularité de délégation par rôle (admin local DC = admin local DHCP par ex si mutualisé)
	5. DANGER : difficulté lors des migration si plusieurs services
2. Système critique : doit être mis à jour en priorité
	1. Sinon, être en capacité de détecter la vulnérabilité non appliquée (via doc d'update)
3. S'assurer de la sécurité physique des DC et systèmes critiques
	1. Si impossible : RODC
4. Gérer les sauvegardes
	1. Faire des sauvegarde
	2. Les protéger
5. Désactiver les protocoles obsolètes (SMBv1 par exemple)
6. Configurer et activer le pare-feu

## Contrer la compromission d'identifiants

Les options :
1. Brute force
2. Password spraying (utiliser mdp commun sur multiple user pour ne pas être détecté)
3. Base de données de comptes compromis
4. Ecouter le traffic (attaquant deja dedans, et auth en clair sur le réseau)
5. Keylogger
6. Phising

---

Déterminer comptes à privilèges
1. 2 comptes : normal et administrateur (restrictions spécifiques)
2. Quels sont les actions d'administration qui ont un impact sur la sécurité ?
	1. Modification de compte sensible
	2. Se connecter sur un système critique
	3. Executer du code sur un DC
3. Quels sont les groupes à protéger :
	1. Groupes protégés par "adminSDHolder"
		1. Situé dans le conteneur system
		2. Permissions de l'adminSDHolder = Permission écrite sur tous les objects groupe protégé par SDHolder et leur contenu
		3. Pourquoi ? Protéger les compte à privilège peu importe quel est leur conteneur
		4. RAPPEL : un object hérite des permissions de son conteneur PARENT
		5. EXEMPLE : admin déplacé dans OU helpDesk, sans SDHolder, les membres helpdesk peuvent réinitialiser son password
		6. Qui est protégé (group) ?
			1. Account / Backup / Print /Server Operator
			2. Administrators
			3. Domain Admins
			4. Enterprise ADmins
			5. Schema ADmins
			6. Read-Only Domain Controllers
			7. Replicator
			8. Enterprise/Key Admins (windows 2019 mini)
			9. Administrator (user)
			10. krbtgt (user)
			11. Domain Controllers (exception : group protégé mais pas contenu)
		7. Toutes les 60 min -> PDC emulator execute process qui énum grpupe protégé + contenu -> analyse descripteur de sécurité VS permission de SDHolder -> Si non correspondance FLAG + UPDATE
	2. Certains groupes sensibles non protégés :
		1. DNSAdmin ( droit executer dll sur DC = injection dll)
		2. Group Policy Creator Owners
		3. Incoming Forest Trust Builder
		4. Remonte Desktop Users



