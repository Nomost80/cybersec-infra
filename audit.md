Norme cadre audit : ISO 19011

il faut prouver la faille (capture d'écran) et expliquer le scénario/procédure.

1. Preuve d'audit
2. Critère d'audit
3. Programme d'audit 
4. Plan d'audit (activités)
5. Champ de l'audit (étendue et limite

Il faut nommer un responsable d'audit

Plan d'audit :
* objectif (blackbox / whitebox ?, review de fichiers de conf ?) => plutôt greybox, si on est bloqué on va demander du jeu
* le périmètre (unité orga, fonctionnel, process)
* justifier pourquoi on a pris tel auditeur (profil, genre cv en annexe), qui va intervenir sur quoi
* le délai convenu pour faire cette partie, soi faut réduire le champ ou la profondeur (date, lieu, planif des réunions)
* Revue documentaire (document qu'on va fournir et qu'on nous a livré)
* Identifier les risques de l'audit (comme les interruptions de service)
* Méthodes
* Ressources

Responsable d'audit :
* vérif le périmètre, délai
* suivi et collecte des preuves d'audit
* constat d'audit
* rapport de non conformité (Il faut expliquer pourquoi y'a une vuln et dire comment la corriger)
* il faut notifier les écarts (changement dans l'archi, de périmètre)

Il faut dire aussi ou seront stockés les données de l'audit et une preuve/comment de leur suppression

Si on détecte une vuln critique on est censé prévenir le client tout de suite.

Rapport d'audit : 
* Introduction : périmètre, limites...., s'engager sur la confidentialité (mettre un tag confidentiel)
* Synthèse des vulnérabilités, il faut définir différentes échelles.
* Observations : synthèse des observations (libellé, risque, niveau de criticité, impact, exploitabilité, droit nécessaire, complexité de corection supposée), synthèse des recommandations (recommandation, priorité, complexité supposée), synthèse des scénarios d'attaque, fiches scénario (schéma, critique, impact, exploitabilité, droits nécessaire, prérequis, explication du scénario + preuve photo, recommandation)
* Observations Détaillées (tous les preuves d'audit si elle ne tienne pas dans le scénario)
* Annexes (caractérisation des indicateurs (criticité, exploitabilité, complexité, impact : disponibilité, intégrité, confidentialité, preuve = altération de la tracabilité), matrices de risques)

Note: Faut que ce soit visuel, codes couleurs...
Il faut bien mesurer les risques. Il faut préciser les outils