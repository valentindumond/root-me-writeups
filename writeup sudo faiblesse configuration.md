sudo - faiblesse de configuration — App-Script — Facile

Plateforme : Root-Me Lien Date de résolution : 24/09/2026

Contexte

Le challenge fournit un accès SSH direct (identifiants dans l'énoncé) et un dossier personnel contenant un fichier readme.md indiquant l'objectif : lire un fichier .passwd situé dans /challenge/app-script/ch1/ch1cracked/, dossier appartenant à un autre utilisateur. L'énoncé lui-même ("cet administrateur n'a pas pensé aux effets de bord en ne modifiant pas les droits") et la ressource associée ("sudo you are doing it wrong") orientent directement vers une règle sudo mal configurée.

Reconnaissance

Première tentative d'accès direct au dossier cible :

bash
cat ch1cracked
# Permission non accordée
ls -la ch1cracked
# Permission non accordée

Sans surprise, aucun accès direct n'est possible avec les droits par défaut de app-script-ch1. L'indice pointant vers sudo, j'ai vérifié les droits sudo accordés à mon utilisateur :

bash
sudo -l

Résultat :

(app-script-ch1-cracked) /bin/cat /challenge/app-script/ch1/notes/*

Cette ligne autorise app-script-ch1 à exécuter /bin/cat avec les droits de app-script-ch1-cracked, mais uniquement sur des fichiers situés dans /challenge/app-script/ch1/notes/.

Découverte de la vulnérabilité

Ma première hypothèse a été d'exploiter le * comme un glob shell classique, en créant un lien symbolique dans notes/ pointant vers le fichier .passwd cible. Cette piste s'est révélée impossible : le dossier notes/ n'est pas accessible en écriture pour app-script-ch1 (dr-xr-x--x), donc aucune création de fichier n'y est possible.

La vraie faille est ailleurs : le * de la règle sudoers n'est pas un glob interprété par le shell avant l'appel à sudo — c'est sudo lui-même qui compare l'argument final à ce motif, selon les règles de correspondance de fnmatch(). Contrairement à un glob shell strict, ce * matche également le caractère /, ce qui permet d'inclure une séquence ../ dans le chemin fourni en argument : la chaîne reste conforme au motif autorisé du point de vue de sudo, tout en ciblant en réalité un fichier situé hors du dossier notes/.

Comme la commande tapée ne contient aucun caractère de glob (*, ?, etc.), le shell local ne modifie rien avant l'exécution : le chemin complet, incluant ../, est transmis tel quel à sudo.

Exploitation

Une seule commande suffit, en utilisant un ../ pour remonter d'un niveau depuis notes/ et rejoindre le dossier ch1cracked/ :

bash
sudo -u app-script-ch1-cracked /bin/cat /challenge/app-script/ch1/notes/../ch1cracked/.passwd

sudo valide que l'argument correspond bien au motif notes/*, puis /bin/cat, exécuté avec les droits de app-script-ch1-cracked, résout le chemin réel après traversée du ../ et affiche le contenu de .passwd.

Résultat

La commande a directement affiché le contenu du fichier .passwd, correspondant au mot de passe de validation du challenge. Le flag a été soumis sur la page Root-Me sans être republié ici, conformément aux règles de la plateforme.

Ce que j'ai appris

Ce challenge montre qu'une règle sudoers limitée par un chemin avec un joker (*) n'offre pas la garantie qu'on croit intuitivement : le motif est évalué comme une correspondance de motif de bas niveau (proche de fnmatch), pas comme un glob shell restreint au dossier courant. Un ../ dans l'argument reste conforme au motif tant qu'il matche textuellement, ce qui permet de sortir du répertoire visé. La leçon directement réutilisable en audit : toujours tester les règles sudoers se terminant par un wildcard avec des séquences de traversée de répertoire (../), et recommander côté défense l'usage de chemins absolus complets sans joker, ou l'option secure_path/validation stricte des arguments plutôt qu'un simple motif de fin de chemin.

Outils utilisés
SSH (client OpenSSH)
sudo -l (audit des permissions sudo)
Bash (construction du payload de traversée de répertoire)