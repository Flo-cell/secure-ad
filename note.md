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
<code>Get-Module -Name NetCease | Format-List</code>
<code>Get-NetSessionEnumPermission | Out-GridView</code>
<code>Set-NetSessionEnumPemission</code>
Necessite reboot du serveur<br>
A inclure en stratégie globale, sur l'ensemble des serveurs 445 SMB


