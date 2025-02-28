# Hardenning_OS4

## 0. Prérequis

➜ **Une seule VM Rocky**

- une seule fera le taff, on se concentre sur le fonctionnement de l'OS en lui-même aujourd'hui encore

🌞 **Installer NGINX**

- ce sera fait ! Ce sera pratique pour nos premiers exemples
- n'oubliez pas d'ouvrir le port firewall
- dans le compte-rendu : un `curl` depuis votre PC pour prouver que le site est dispo
```
[dash@localhost ~]$ sudo dnf install -y nginx
[dash@localhost ~]$ sudo systemctl enable nginx
[dash@localhost ~]$ sudo systemctl start nginx
[dash@localhost ~]$ sudo systemctl status nginx
[dash@localhost ~]$ sudo firewall-cmd --add-port=80/tcp --permanent
[dash@localhost ~]$ sudo firewall-cmd --reload
```

🌞 **Dans la conf NGINX par défaut**

- y'a une conf pour écouter sur le port 80, et qui sert la page d'accueil par défaut
- mettez en évidence ces lignes (vous cherchez un bloc `server {}`)

```
[dash@localhost ~]$ nano /etc/nginx/nginx.conf
server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }
```
Le chemin est le suivant `/usr/share/nginx/html`


# Part I : Premiers pas

## 0. Intro

➜ **SELinux est un module du kernel**

Vous ne verrez pas de processus SELinux s'exécuter, il est **dans** le kernel.

> Un "module", c'est un "plugin" ou un "mod" si vous préférez : un truc qui rajoute des fonctionnalités. En l'occurrence, un module kernel : il rajoute des fonctionnalités au kernel.

➜ **SELinux met en place du contrôle d'accès**

Dès qu'un programme essaie d'interagir avec l'OS (lecture ou écriture d'un fichier, lancer un nouveau programme, utiliser le réseau, etc.), SELinux peut décider si l'action est autorisée ou non.

SELinux s'ajoute à la gestion de contrôle d'accès habituelle (users + groupes + permissions rwx).

➜ **SELinux peut être dans 3 états :**

- **enforcing** : il est activé, il peut bloquer certaines actions, et il log toutes les actions qui enfreignent les règles SELinux définies
- **permissive** : il est activé, mais il log seulement toutes les actions qui enfreignent les règles SELinux définies (il ne bloque rien, il log juste)
- **disabled** : 'lé po activé

➜ **SELinux est un système de labelling**

- on définit un label (un "tag") sur **tous** les fichiers du système : on appelle ça un **contexte SELinux**
  - on peut le voir avec `ls -Z`
- les processus ont aussi un contexte
  - on peut le voir avec `ps -Z`
- on définit un label sur un processus, un label sur un fichier, puis on écrit une règle pour indiquer que le contexte de notre processus est autorisé à interagir avec le contexte du fichier
- il faut donc absolument tout whitelister ! La moindre interaction.

➜ **Ce qui nous intéresse principalement aujourd'hui dans le contexte SELinux, c'est le *type*.**

C'est le troisième champ du contexte SELinux. C'est le truc qui se termine par `_t` par convention. 

C'est par exemple `httpd_sys_content_t` dans `system_u:object_r:httpd_sys_content_t:s0.`

> Pour un processus, le type, SELinux appelle aussi ça un *domain*.

## 1. Enabling SELinux

🌞 **Vérifier l'état actuel de SELinux**

- à votre dispo :
  - la commande `sestatus` affiche l'état actuel de SELinux
```
[dash@localhost ~]$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
[dash@localhost ~]$ sudo setenforce 1
[sudo] password for dash:
[dash@localhost ~]$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```
```
[dash@localhost ~]$ sudo nano /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted
```
- vérifiez qu'il est en mode *targeted* (pas le mode *mls* ou autre)

## 2. The Z flag

➜ Beaucoup de commandes usuelles sont désormais pourvues de l'option `-Z` qui permet d'afficher les labels SELinux.

```bash
# voir les labels sur des fichiers/dossiers/etc.
$ ls -al -Z

# voir les labels sur des users
$ id -Z

# voir les labels des processus en cours d'exécution
$ ps -ef -Z

# voir les labels des ports TCP et UDP en écoute
$ ss -lnptu -Z
```

🌞 **Déterminer le *type* SELinux de...**

- l'utilisateur NGINX
```
[dash@localhost ~]$ sudo -su nginx -s /bin/bash
bash-5.1$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```
- le fichier de conf de NGINX
```
[dash@localhost ~]$ ls -Z /etc/nginx/nginx.conf
system_u:object_r:httpd_config_t:s0 /etc/nginx/nginx.conf
```
- le programme NGINX sur le disque
```
[dash@localhost ~]$ ls -Z $(which nginx)
system_u:object_r:httpd_exec_t:s0 /usr/sbin/nginx
```
- le dossier dans lequel se trouve le fichier HTML par défaut
```
[dash@localhost ~]$ ls -dZ /usr/share/nginx/html
system_u:object_r:httpd_sys_content_t:s0 /usr/share/nginx/html
```
- le fichier HTML par défaut
```
[dash@localhost ~]$ ls -Z /usr/share/nginx/html/index.html
system_u:object_r:httpd_sys_content_t:s0 /usr/share/nginx/html/index.html
```
- le processus NGINX en cours d'exécution
```
[dash@localhost ~]$ ps -Zef | grep nginx
system_u:system_r:httpd_t:s0    root       12606       1  0 15:08 ?        00:00:00 nginx: master process /usr/sbin/nginx
system_u:system_r:httpd_t:s0    nginx      12607   12606  0 15:08 ?        00:00:00 nginx: worker process
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 dash 12772 1016  0 16:00 pts/0 00:00:00 grep --color=auto nginx
```
- le port TCP utilisé par NGINX pour écouter
```
[dash@localhost ~]$ sudo semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```


## 3. Voir les règles définies

Pour voir des règles, on utilise généralement `sesearch`.

Utilisation typique :

```bash
sesearch --allow --source un_truc_t --target un_truc_t --class une_classe

# par exedmple
sesearch --allow --source httpd_t --target httpd_t --class file
```

🌞 **Afficher les règles liées à NGINX**

- afficher les règles que le type SELinux du processus NGINX...
- ...a sur le type SELinux du fichier HTML
```
[dash@localhost ~]$ sesearch --allow --source unconfined_t --target httpd_config_t --class file
allow domain file_type:file map; [ domain_can_mmap_files ]:True
allow files_unconfined_type file_type:file execmod; [ selinuxuser_execmod ]:True
allow files_unconfined_type file_type:file { append audit_access create execute execute_no_trans getattr ioctl link lock map mounton open quotaon read relabelfrom relabelto rename setattr swapon unlink watch watch_mount watch_reads watch_sb watch_with_perm write };
```


# Part II : Un peu de conf

On va à chaque fois procéder de façon itérative :

- modifier la conf
- redémarrer le service
- constater le *sproutch*
- afficher les logs SELinux
- trouver la ligne de log qui nous dit exactement ce qui a été bloqué

## 1. Racine web

Vous l'avez normalement repéré en intro, NGINX sert une page d'accueil par défaut. Elle est dans `/usr/share/nginx/html` (comme indiqué dans le fichier de conf).

On va déplacer ça dans `/var/www/meow/`.

**C'est un bon prétexte pour voir l'interaction entre un processus et un fichier lorsque SELinux est activé.**

🌞 **Créez-moi tout ça**

- un dossier `/var/www/meow/` qui appartient au user qui lance NGINX
- ontient un fichier `index.html` (contenu de votre choix) qui appartient au user qui lance NGINX
- droits de lecture sur le dossier/le fichier pour le propriétaire

🌞 **Modifier la conf NGINX**

- pour que le site servi sur le port 80 ne soit plus celui de `/usr/share/nginx/html` mais celui qui est dans `/var/www/meow/` (une seule ligne de conf à modifier)
- (re)démarrez le service  `nginx`
- visitez la page web et constater le sproutch (403)


🌞 **Logs !**

- repérez la ligne de log qui montre l'interaction qui a été bloquée
- le fichier de log c'est `/var/log/audit/audit.log`
- vous cherchez une ligne qui contient le mot `avc` (c'est le nom des blocages SELinux)
- et qui contient le *type* du processus `nginx` que vous avez repéré plus tôt

> La ligne doit explicitement mentionner un blocage pour la lecture d'un fichier.

🌞 **Etat des lieux**

- afficher le contexte SELinux de `/usr/share/nginx/html` (oui tu l'as déjà fait, je sais)
- afficher le contexte SELinux de `/var/www/meow`
- constater qu'ils sont différents, et que ça sent le vinaigre

> Il faudrait que notre processus NGINX puisse accéder à ce dossier. Plein de façons de faire : on pourrait par exemple créer un nouveau type `meow_t`, l'attribuer à notre fichier `index.html` et autoriser le *type* du processus NGINX à le lire. On va rien faire de tout ça :D

🌞 **Conf simpliste**

- on va se contenter d'appliquer à notre `/var/www/meow/` la même conf que le dossier de base
- je vous file la commande :

```bash
# copier récursivement les contextes SELinux d'un dossier vers un autre
chcon -R --reference /usr/share/nginx/html /var/www/meow
```

🌞 **Constater le changement**

- votre dossier `/var/www/meow` et son contenu devraient avoir un nouveau contexte SELinux

🌞 **Redémarrez NGINX**

- visitez le site web
- no sproutch ?

## 2. Port

Idem, toujours au même endroit dans la conf, vous l'avez repéré en intro, NGINX écoute par défaut sur le port 80.

On va changer ça pour un autre port non-conventionnel : 8888/tcp.

**C'est un bon prétexte pour voir l'interaction entre un processus et un port TCP lorsque SELinux est activé.**

🌞 **Modifier la conf de NGINX**

- il doit écouter sur le port 8888/tcp
- n'oubliez pas d'ouvrir ce port dans le firewall
- rédémarrez NGINX après avoir modifié la conf
- constater un sproutch immédiat au redémarrages

🌞 **Logs logs logs !**

- repérez la ligne de log qui montre l'interaction qui a été bloquée
- vous cherchez toujours une ligne qui contient le mot `avc` (c'est le nom des blocages SELinux)
- et qui contient le *type* du processus `nginx` que vous avez repéré plus tôt
- et qui mentionne explicitement un blocage sur le port TCP (tcp socket) 8888

➜ **On va procéder différemment pour le port**

- on va continuer à réutiliser la conf existante
- il existe déjà une liste de ports qui portent le type `http_port_t` par défaut
- le type de NGINX a le droit d'écouter sur les ports `http_port_t` par défaut

🌞 **Marquons le port `8888/tcp` avec le type `http_port_t`**

- la commande :

```bash
semanage port -a -t http_port_t -p tcp 8888
```

- prouvez que votre port est bien dans la liste des `http_port_t` avec

```bash
semanage port -l
```

🌞 **Redémarrez NGINX**

- no sproutch ?

## 3. Your own policy

Actuellement, SELinux a une *policy* chargée : un ensemble de règles (des kilotonnes déjà sur une install de base de Rocky) qui détermine ce qui est autorisé pour énormément d'applications qu'on peut installer via les paquets.

C'est modulaire comme truc : on écrit un fichier de conf SELinux par programme, et tout est compilé en une *policy* unique.

Vous pouvez lister les modules chargés dans la policy actuelle avec :

```bash
semodule -l
```

Bon, et on peut nous-mêmes écrire un fichier de règle SELinux, et en faire un module, et l'ajouter à la *policy*. Idéal si on a un super service fait maison, et qu'on souhaite ajouter une policy pour lui !

Genre j'sais pas, UNE CALCULATRICE RESEAU.

➜ **Récupérez la calculatrice réseau**

- le fichier de code + votre `.service`
- avec SELinux activé, le service ne devrait pas démarrer

➜ **Là encore on va rester simple, et utiliser une technique différente**

- quand une action est bloquée, ça produit une ligne de log dans `/var/log/audit.audit.log` qui explique précisément ce qui a été bloqué (on peut utiliser la commande `ausearch` pour chercher facilement avec options cools dans ce fichier)
- on peut générer une conf qui autorise cette action, à partir de la ligne de log, avec la commande `audit2allow`
- je recommande des trucs du genre `sudo ausearch -m AVC -ts recent | tail -n200 | sudo audit2allow -a -m meow` pour générer automatiquement un module pour une policy SELinux :
  - n'affiche que les logs récents
  - récupère les 200 dernières lignes
  - produit la conf avec `audit2allow`
  - le module sera nommé `meow`

> On va pas faire du die&retry : lancer le truc, générer une ligne de nouvelle conf, relancer, ça crash encore, on regénère une ligne, et ainsi de suite. Le mode permissive de SELinux est là pour ça : il génère tous les logs, sans nous bloquer.

![audit2allow](./img/audit2allow.png)

🌞 **Passer temporairement SELinux en mode *permissive***

- avec un `sudo setenforce 0`
- vérifier avec un `sestatus`

🌞 **Lancer l'application**

- avec un `sudo systemctl restart calculatrice`
- elle devrait fonctionner

🌞 **Observer les logs**

- vous devriez voir des trucs bloqués en relation avec notre service
- avec un :

```bash
sudo ausearch -m AVC -ts recent | tail -n200
```

🌞 **Observer la conf autogénérée**

- même commande, mais on rajoute `audit2allow`, go faire : 

```bash
sudo ausearch -m AVC -ts recent | tail -n200 | sudo audit2allow -a -m meow
```

🌞 **Stocker la conf générée**

- on redirige le tout dans un fichier qui porte l'extension `.te` par convention
- go :

```bash
# allez dans votre homedir
cd

# générez le fichier de conf pour un nouveau module SELinux
sudo ausearch -m AVC -ts recent | tail -n200 | sudo audit2allow -a -m meow > meow.te
```

🌞 **Appliquer la conf**

- on va compiler ce nouveau module SELinux `meow`
- et on pourra ensuite le charger dans notre policy SELinux actuelle
- suivez le guide :

```bash
# toujours dans le même dossier, avec le fichier meow.te

# on compile le module en un .pp
sudo checkmodule -M -m -o meow.mod meow.te
sudo semodule_package -o meow.pp -m meow.mod

# chargement du module dans notre policy actuelle
# ça peut prendre un peu de temps
sudo semodule -i meow.pp
```

🌞 **Repasser SELinux en mode *enforcing***

- avec un `sudo setenforce 1`
- vérifier avec `sestatus`

🌞 **Redémarrer le service**

- shoud work !

🌞 **Observer le nouveau module chargé**

- lister les modules SELinux en cours de fonctionnement
- et `grep meow` !

![shell_as_root](./img/shell_as_root.png)
