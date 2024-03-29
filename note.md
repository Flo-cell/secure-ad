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

LDAP expose l'annuaire et permet l'énumération de compte :
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

**Enumération SMB** :<br>
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

**Enumération SAM-R :**<br>
1. Outil : SuperScan 4 par exemple
2. Vu précédement pour anonyme, à désactiver via groupe built-in "Pre windows 2000 Compatible Access)
3. Restreindre l'API SAM à des groupes particuliers :
	1. Via GPO : Accès réseau : restreindre les clients autorisés à effectuer des appels distants vers SAM
	2. Ne pas appliquer sur les DC !

---

**Recommandations** :
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

**Les options :**
1. Brute force
2. Password spraying (utiliser mdp commun sur multiple user pour ne pas être détecté)
3. Base de données de comptes compromis
4. Ecouter le traffic (attaquant deja dedans, et auth en clair sur le réseau)
5. Keylogger
6. Phising

---

**Déterminer comptes à privilèges :**
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

---

**Renforcer la protection des comptes à privilège**

1. Stratégie local
	1. Ensemble de paramètres de sécurité
	2. A renforcer via GPO
2. Stratégie de groupe
	1. Imposer les paramètres de manière globale
3. Outils de gestion
	1. Microsoft Security Compliance Toolkit
	2. ADACLScanner

---

**Stratégie de mot de passe**

1. Mot de passe = clef symétrique générée par un humain
2. FGPP (Fine Grained Password Policies)
3. Paramètres recommandées :
	1. longueur minimale > 14
	2. Compléxité : activée
	3. Durée max : 30 et 90 jours
4. Vérrouillage de comtpe :
	1. Empeche pas : Brute force / password spray / vol ID
	2. 1 tentative auth != 1 tentative user
	3. Seuil dépend de la détection de l'entreprise
5. Azure AD Password Protection
	1. Protection password spray (empeche choisir mot commun en pass)
	2. Azure AD premium P1 (fonctionalité sur site)

---

**S'assurer qu'un compte n'est pas compromis**

1. Azure AD Identity Protection
2. Azure AD premium P2
3. Azure AD Connect Password Hash Synchronization option
4. Aucun moment le mot de passe est partagé avec Azure AD

---

**Sécuriser les authentifications LDAP**

1. Empecher les DC de répondre à une demande LDAP avec password non chiffré
2. Bind request
3. GPO : Controleur de domaine : conditions requises pour la signature de serveur LDAP / Exiger la signature
4. DANGER : mot de passe quand même envoyé en clair mais DC la refuse
5. Utiliser LDAP sécurisé : port 636

---

**Protocoles obsolètes**

1. LM : 1993
2. SMBv1 : 2006
3. NTLMv1 : 1998
4. Kerberos 3DES : 2000
5. Wdigest : 1994
6. Existante du groupe "Protected User" pour bloquer ces vieux protocoles

---

**Kerberos Roasting**

1. Trouver mot de passe sans tentative d'auth : Kerberos roasting
2. GPO : Sécurité réseau : Configurer les types de chiffrement autorisés pour Kerberos (retirer au moins DES)
3. User / Propriété / Ne pas cocher dans compte : Utiliser...DES via Kerberos / Cocher AES 256
4. Si compte de service : donner un gMSA
5. Ne jamais cocher : La pré auth Kerberos n'est pas nécéssaire
	
---

**Sécurisation contre les mouvements latéraux**

1. Abuse de la fonctionnalité de SSO
2. Comment ça marche ?
	1. winlogon.exe collecte le secret à l'ouverture de session
	2. Il passe enssuite au processus LSASS (cerveau de sécurité Windows)
	3. LSASS authentifie le user et met en cache les secrets
	4. Si attaquant à privilège de debug "SeDebugPrivilege" il peut extraire les infos de LSASS
	5. Via mimikatz
3. Il faut considérer un compte log sur un PC comme compromis
4. Comment ne pas exposer ces ID lorsqu'on se connecte ?
	1. Utiliser la fonction "/RestrictedAdmin" => mstsc /RestrictedAdmin
	2. Restriction : Pas d'identité sur le réseau si ce n'est celle de la machine elle même
	3. Utiliser la fonction "/RemoteGuard" => mstsc /RemoteGuard (conserve son identité)
	4. Restriction : server en face doit être compatible remote guard
5. Recommandations :
	1. Désactiver WDigest (via GPO) et monitorer sa réactivation
	2. Utiliser les modes sécuritaires de RDP (RestrictedAdmin et RemoteGuard)

---

**Limiter ou les systemes/comptes peuvent se connecter**

1. Pre server 2012 : GPO (interdire l'accès à cet ordinateur à partir du réseau / Interdire l'ouverture ...)
2. A partir de 2012 :
	1. Stratégies d'authentification et silos (centre d'administration Active Directory)
	2. Défini les critères pour que l'obtention d'un TGT soit valide
	3. Incontournable meme en admin local
	4. Visualiser son silo : whoami /claims
	5. Evenement généré pour surveillance (non actif par defaut)

---

**Sécuriser les comptes locaux**

1. Microsoft LAPS
2. On peut interdir un compte local d'être utilisé à travers le réseaux (le support local pourra se co)
3. Pare feu windows => Qui peut établir des connections entre machines

---

**Les machines d'administration**

1. PAW : Privilege Access Workstation
2. Machine physique dédiée
3. Restrictions réseau / de compte / outils de surveillance
4. Machine jetable (facilement redéployable)
5. Serveur de rebond utile, mais seulement si on rebondis depuis des PAW
6. Connection à une machine d'administration depuis une machine non dédié n'offre aucune sécurité

---

**Délégation Kerberos**

1. Kerberos forwarding :
	1. User => Front end => Back end
	2. Si kerberos Forwarding : on autorise front end à demandé ticket au back end pour l'user
	3. Le back end pense que la demande provient directement de l'user
2. Front end compromis : on peut se faire passer pour l'user partout
3. Il existe des options pour limiter cela :
	1. Dans AD : User / compte / Le compte est sensible et ne peut être délégué
	2. Délégation contrainte : Propriété / User / Délégation 'n'approuver..."

---

**Relation d'approbation**

1. Pas de compte de domaine approuvée dans vos groupes d'admins
2. Activer AES dans le compte relation d'approbation
3. KBSO
4. Authentification sélective

---

## Attaques post-exploitation

**Domination du domaine**

1. Pourquoi ?
	1. Etendre son périmètre
	2. Déployer malware via GPO
	3. Rester persistant
2. Répliquer admin directory pour extraire (attaque DCSync)
	1. Outils : DSinternal
	2. Get-ADReplAccount -All -Server DC01 (replication totale)
	3. Get-ADReplAccount -SamAccountName krbtgt -Server DC01 -Domain CONTOSO
3. Extraire compte krbtgt (kerberos)
	1. Générer ses propres tickets (sans être admin)
	2. mimikatz golden ticket (/ptt PASSTHETICKET)
4. Changer mot de passe krbtgt ? Utile ?
	1. 1er changement : ancien marche
	2. 2eme changement : ancien ne marche plus
	3. ODD (oportunité detection) : log KRB_AP_ERR_BAD_INTEGRITY (31)(4769 : code erreur 1F) (utilisation vieux ticket kerberos)
5. Extraire la base de données AD
	1. Outils à utiliser : wmic
	2. Se trouve dans c://Windows/NTDS/ntds.dit

---

## Détecter les activités suspectes

**Défis liés à la détection**

1. Configurer sa stratégie d'audit en limitant bruit et minimum de rétention
2. Stratégie ordinateur local / conf ordinateur /param windows / param sécu / config avancée de la stratégie d'audit / stratégie audit système
3. Connaitre la stratégie appliquée : c:\>auditpol /get /category:*
4. Activer : Audit de création de processus

---

**Microsoft defender for identity**

1. Permet de détecter toutes les attaques vues
2. Necessite licence E5
3. Peut envoyer sur SIEM On Premise

---

## Points à retenir

1. Prévenir les actions de reconnaissance en limitant l'énumération
2. Pas de mutualisation de services, patching rapide, accès physique, sauvegarde à protéger
3. Eviter mouvements latéraux
4. 2 comptes différents pour administration et bureautique (interdire ouverture de session entre tiers)
5. Définir qui peut faire quoi ?
6. Utiliser azure AD password protection et Azure AD identity protection
7. Réduire exposition comptes (pas de mise en cache)
8. Utiliser LAPS (mot de passe local)
9. Prévoir des machines dédiées à l'administration
10. Désactiver protocoles obsolètes
