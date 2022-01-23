## Qui controle AD ?

1 - Compte à privilege sur AD
2 - Controleur de domaine (physique)
3 - Dependance de sécurité (SCCM)
4 - Permission sur objet (permet privilege)

## Principe de tier

Tier 0 : Système critique (serveur AD, serveur certificat, ADFS, système avec dépendance sécurité, poste administration de système critique)
Tier 1 : Serveur fichier et autres serveurs (propres compte pour l'administration)
Tier 2 : Station de travail

## Chronolgie de l'attaque

1 - Préparation et reco
2 - Première ressource compromise
3 - Compte administrateur compromis (24h à 48h)
4 - Détection de la compromission (200 jours)

**But du jeu :**  Rendre la partie 2 longue et difficile, rendre non rentable l'attaque\
Le détecter le plus vite possible


