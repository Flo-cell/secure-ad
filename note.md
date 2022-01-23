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
	5. 



