Avec l'avènement de Docker, le *prêt à déployer* n'a jamais été aussi facile d'approche. Aujourd'hui, je vous propose de monter une seedbox avec Docker, accompagnée de quoi égayer vos journées à ne rien faire de productif.

***DISCLAIMER : le piratage c'est mal, et n'oubliez pas de manger vos 5 fruits et légumes par jour.***

### Les prérequis
Pour commencer, l'essentiel serait de disposer d'un serveur ainsi que d'un nom de domaine. Si le serveur n'est pas très puissant, il faudra faire l'impasse sur le *transcoding* (ça n'impactera que Emby).

Ça peut paraître évident, mais je pense qu'avoir des notions en matière de conteneurs et de para-virtualisation fera toujours le plus grand bien. En fait, il suffit juste de savoir un peu comment Docker fonctionne. Car peut-être qu'aujourd'hui ce sera un peu magique pour vous, mais le lendemain quand vous aurez à déboguer un problème, il ne faudra pas venir chialer parce qu'on ne comprend rien et que *rien ne marche*. Commencez par exemple à en savoir un peu plus sur Docker, [ici](https://en.wikipedia.org/wiki/Docker_%28software%29) et [là](https://docs.docker.com/).

![](https://pix.schrodinger.io/ldJb6tXB/0aHpFyS1.png)

Avoir Docker d'installé sur sa machine aide aussi... Supposons que vous utilisez une Debian Jessie, les manipulations sont plutôt courtes. Commencez par ajouter les dépôts nécessaires (je préfère avoir la dernière stable plutôt qu'une vieille version de Debian, Docker évoluant très vite) :

```language-bash
aptitude install apt-transport-https ca-certificates
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo "deb https://apt.dockerproject.org/repo debian-jessie main" > /etc/apt/sources.list.d/docker.list
aptitude update
aptitude install docker-engine
service docker start
```

Voici pour une installation express, je ne prendrai pas la peine d'aborder les subtilités. Je vous recommande [cet excellent tutoriel](https://mondedie.fr/viewtopic.php?id=7164) pour cela, tutoriel qui vous fera faire rapidement une prise en main de Docker.

*docker-compose* est un outil très simple et puissant pour gérer des conteneurs et éviter de devoir noter quelque part de longs `docker run`. On va l'utiliser, donc il vaudrait mieux l'installer :

```language-bash
aptitude install python-pip
pip install docker-compose
```

### rTorrent/ruTorrent
Au cas où vous le ne sauriez pas (encore), rTorrent est un client BitTorrent reposant sur libtorrent, tandis que ruTorrent est une interface utilisateur accessible avec un navigateur Web. Le choix de rTorrent s'impose pour les utilisateurs les plus exigeants.

Maintenant, il va falloir compiler libtorrent,  rTorrent, mediainfo ; puis télécharger ruTorrent, lui installer des plugins, un joli thème ; et enfin il faudra tout configurer à la main. Temps estimé : environ 2 heures, avec des cheveux en moins. Nan, je déconne. En fait on va utiliser une image Docker qui fait tout ça pour nous, qui est déjà configurée... prête à être lancée en fait, aussi dur à croire que cela puisse paraître : **wonderfall/rutorrent**, et elle est de ma propre conception. Je l'ai forkée il y a un moment du projet de *xataz* que je remercie au passage.

**Features :**

- Basée sur [Alpine Linux](http://alpinelinux.org/), donc relativement légère.
- [rTorrent](https://rakshasa.github.io/rtorrent/), [libtorrent](https://rakshasa.github.io/rtorrent/), [mediainfo](https://mediaarea.net/en/MediaInfo) : dernières versions en date.
- [ruTorrent](https://github.com/Novik/ruTorrent.git) avec plugin [ruTorrentMobile](https://github.com/xombiemp/rutorrentMobile.git).
- PHP 7 (ruTorrent) sous le capot !
- Thèmes disponibles : [Material](https://github.com/Phlooo/ruTorrent-MaterialDesign.git) (par défaut) + [FlatUI](https://github.com/exetico/FlatUI.git) (Dark & Light) 
- [Filebot](http://www.filebot.net/) pour un traitement automatique des films et séries.
- Configuration optimale et sécurisée qui conviendra au plus grand nombre.
- Fonctionnelle dès le déploiement.

Les sources sont disponibles [ici](https://github.com/Wonderfall/dockerfiles/tree/master/rutorrent) ; l'image est disponible sur le [Docker Hub](https://hub.docker.com/r/wonderfall/rutorrent/), et je n'ai pas jugé utile de lui faire des tags. C'est une *automated build* donc Docker Hub récupère tout seul le Dockerfile depuis les sources puis construit l'image avec.  Voici à quoi ça ressemble visuellement :

![rutorrent](https://pix.schrodinger.io/KDVxwnJA/nEMCzJEd.jpg)
*(Je seedais des ISOs GNU/Linux.)*

### Emby
Emby est une alternative *libre* à PMS (Plex Media Server). Emby peut détecter vos films, séries et musiques et les rassembler dans des bibliothèques. Il s'occupe également de télécharger les bonnes métadonnées. Emby dipose d'une interface Web à partir de laquelle vous pouvez lire en *streaming* vos médias. Emby permet bien évidemment l'usage du *transcoding*. Une image Docker est disponible et elle a été revue assez récemment, elle est de bonne qualité : [emby/embyserver](https://hub.docker.com/r/emby/embyserver/).

![emby](https://pix.schrodinger.io/z4vqVxKI/Y0TzfZSO.png)

### Subsonic
Subsonic est un logiciel libre disposant d'une interface Web, grâce auquel vous pouvez lire en *streaming* vos musiques (de nombreux clients sont compatibles, et inutile de dire que Subsonic supporte le *transcoding*). Je le destine à mes musiques seulement, en effet je le préfère à Emby pour cette tâche. Je maintiens une image basée sur Alpine Linux : [wonderfall/subsonic](https://hub.docker.com/r/wonderfall/subsonic/).

![subsonic](https://pix.schrodinger.io/3dh0Fnwg/tltP5Eqo.png)

### Le reverse proxy
En fait, on va lancer tous les conteneurs derrière un reverse proxy HTTP, et ce sera nginx, avec mon image [wonderfall/nginx](https://hub.docker.com/r/wonderfall/nginx/), bien que vous pouvez utilisez un autre conteneur disposant d'nginx, ou même continuer à utiliser nginx sur l'host (dans ce cas [cet article](https://wonderfall.xyz/dns-nom-conteneur/) pourrait vous intéresser).

![nginx](https://pix.schrodinger.io/aqluWFZ8/oHNFZMjy.png)

### Installation
Maintenant qu'on a assez blablaté, on peut procéder à l'installation. Je vous recommande de préparer un dossier réservé aux volumes qu'utiliseront les conteneurs, par exemple `/home/docker`. 

Placez-vous dans ce répertoire et créez un dossier compose, dans lequel vous allez pouvoir créer un fichier `docker-compose.yml` avec votre éditeur de texte favori, et dont le contenu est le suivant :

```language-yaml
nginx:
  image: wonderfall/nginx
  container_name: nginx
  environment:
    - UID=1000
    - GID=1000
  ports:
    - "80:8000"
    - "443:4430"
  links:
    - rutorrent:rutorrent
    - emby:emby
    - subsonic:subsonic
  volumes:
    - /home/docker/nginx/sites:/sites-enabled
    - /home/docker/nginx/conf:/conf.d
    - /home/docker/nginx/passwds:/passwds
    - /home/docker/nginx/log:/var/log/nginx
    - /home/docker/nginx/certs:/certs

rutorrent:
  image: wonderfall/rutorrent
  container_name: rutorrent
  environment:
    - WEBROOT=/
    - UID=1000
    - GID=1000
  ports:
    - "49184:49184"
    - "49184:49184/udp"
  volumes:
    - /home/docker/seedbox:/data
    - /home/docker/seedbox/rutorrent:/var/www/torrent/share/users

emby:
  image: emby/embyserver
  container_name: emby
  environment:
    - APP_UID=1000
    - APP_GID=1000
  volumes:
    - /home/docker/seedbox:/data
    - /home/docker/emby:/config

subsonic:
  image: wonderfall/subsonic:6
  container_name: subsonic
  volumes:
    - /home/docker/seedbox/torrents/musics:/musics
    - /home/docker/subsonic:/data
  environment:
    - GID=1000
    - UID=1000
```

Donc en fait le fichier indique que nous voulons lancer 4 conteneurs : nginx, rutorrent, emby et subsonic. Concernant `environment`, il va falloir changer le `UID` et le `GID` pour qu'ils correspondent à l'utilisateur qui détient `/home/docker` (en fait c'est pas la raison, mais c'est dans un souci d'homogénéité entre les applications). 

Vous pouvez garder les volumes (indispensables pour la persistance des données) tels qu'ils sont paramétrés (je détaille ici la structure des volumes un peu plus) :

- `/home/docker/nginx` : conteneur nginx.
  - `sites` : foutez vos vhosts dedans.
  - `conf` : si besoin.
  - `log` : access/error.log.
  - `passwds` : fichiers d'authentification.
  - `certs` : vos certificats, clés privées.
- `/home/docker/seedbox` : conteneur rutorrent.
  - `Media` : symlinks créés par Filebot.
  - `rutorrent` : préférences de rutorrent. 
  - `torrents` : téléchargements de rtorrent.
  - `.session` : pour rtorrent.
  - `.watch` : fonctionnalité watch.
- `/home/docker/subsonic` : conteneur Subsonic.
- `/home/docker/emby` : conteneur Emby.

Pour télécharger les images en une seule fois (ou pour les mettre à jour plus tard), on peut utiliser docker-compose :

```language
docker-compose pull
```

Avant de se précipiter et de tout lancer, il va falloir qu'on s'occupe de configurer les vhosts du reverse proxy qui doivent correspondre à chaque conteneur. `/home/docker/nginx/sites` devra contenir : `emby.conf`, `rutorrent.conf` et `subsonic.conf`. 

C'est là que je ne peux pas trop vous aider puisque je ne sais pas si vous vous voulez utiliser SSL/TLS, ou si vous préférez des webroots personnalisées (auquel cas pensez à modifier la variable d'environnement `WEBROOT` dans le `docker-compose.yml`) plutôt que des sous-domaines (exemples ci-dessous)... En tout cas, le block `location` devrait ressembler à :

```language-nginx
location / {
    proxy_pass http://conteneur:port;
    proxy_set_header        Host                 $host;
    proxy_set_header        X-Real-IP            $remote_addr;
    proxy_set_header        X-Forwarded-For      $proxy_add_x_forwarded_for;
    proxy_set_header        X-Remote-Port        $remote_port;
    proxy_set_header        X-Forwarded-Proto    $scheme;
    proxy_redirect          off;
}
```

Seule la directive `proxy_pass` sera à adapter :

- `proxy_pass http://rutorrent`
- `proxy_pass http://emby:8096`
- `proxy_pass http://subsonic:4040`

Si vous ne voyez toujours pas comment faire, voici quelques exemples de configuration déjà prêtes dans le cas où vous utilisez un sous-domaine, et SSL/TLS :

- [rutorrent.conf](https://paste.schrodinger.io/?eb18699885912511#Pg3bl7v7sfSBk0gZIWJhePFry1FQv+A9WqmCdaSp438=)
- [emby.conf](https://paste.schrodinger.io/?eaa09f55820f3ecf#4STXoqvzih/ZuzoaguZCgCSr+E8QfxKWDoO+BCybSAg=)
- [subsonic.conf](https://paste.schrodinger.io/?d986641c4640495c#1vEHf4p/a8f6LmSRdPdziGi8PcUVQMqfockDiVT4XPc=)

Vous pouvez aussi vous inspirer de la doc rédigée par Hardware à ce sujet : https://github.com/hardware/mailserver/wiki/Reverse-proxy-configuration

Récemment, j'ai ajouté un utilitaire nommé `ngxproxy` dans l'image qui peut vous simplifier cette tâche (à condition d'avoir déjà lancé le compose ci-dessus avec `docker-compose up -d`). Vous aurez à répondre de plusieurs questions et un vhost sera automatiquement créé et installé, par exemple pour rutorrent :

```language
$ docker exec -ti nginx ngxproxy

Welcome to ngxproxy utility.
We're about to create a new virtual host (AKA server block).

Name: rutorrent
Domain: box.domain.tld
Webroot (default is /): /
Container: rutorrent
Port (default is 80):
HTTPS [y/n]: y
Certificate path: /certs/rutorrent.crt
Certificate key path: /certs/rutorrent.key
Secure headers [y/n]: y
Max body size in MB (integer/n): n

Done! rutorrent.conf has been generated.
Reload nginx now? [y/n]:
```
**Concernant l'authentification sur rutorrent**, vous aurez remarqué que je l'ai mise en place de cette façon dans mon vhost :

```language-nginx
server {
  auth_basic "Who's this?";
  auth_basic_user_file /passwds/rutorrent.htpasswd;
}
```

C'est nécessaire parce que sinon n'importe qui pourra accéder à rutorrent depuis l'extérieur, rutorrent n'étant qu'un simple *front-end* et ne disposant pas d'un moyen d'authentification intégré. Du coup il faut générer un fichier, `rutorrent.htpasswd` qui indiquera à nginx de demander au visiteur un couple utilisateur/password, qui doit être valide pour pouvoir accéder au site (ici ruTorrent). Il s'agit d'une **authentification HTTP**. Pour générer ce fichier, vous pouvez utiliser l'utilitaire `ngxpasswd` fourni avec mon image :

```language
$ docker exec -ti nginx ngxpasswd

Welcome to ngxpasswd utility.
We're about to create a password file.

Name: rutorrent
User: user
Password (leave blank to generate one): blablala

A new password file has been saved to /passwds/rutorrent.htpasswd :
- Service  :  rutorrent
- User     :  user
- Password :  blablala

vhost at /sites-enabled/rutorrent.conf detected.
Add authentication to rutorrent.conf? [y/n]: y
Automatically added, please verify. Otherwise follow these instructions.

Paste this to your vhost in order to enable auth :
        auth_basic "Who's this?";
        auth_basic_user_file /passwds/rutorrent.htpasswd;

Reload nginx now? [y/n]:
```

Si vous n'êtes pas inspiré pour le mot de passe, pas la peine de le renseigner, `ngxpasswd` peut en générer un tout seul comme il devrait vous l'indiquer. L'output de la commande vous indique ce que vous devez rajouter dans le vhost de rutorrent s'il ne peut pas l'ajouter tout seul.

Une fois que les fichiers de configuration sont prêts dans `/home/docker/nginx/sites`, il ne reste plus qu'à tout lancer avec un :

```language
docker-compose up -d
```

Et vérifier que *tout va bien dans le meilleur des mondes* avec :

```language
docker-compose ps
```

Si un conteneur ne s'est pas lancé comme il aurait dû le faire, vous pouvez consulter les erreurs avec :

```language
docker logs nom-du-conteneur
```

### Enjoy
rutorrent est déjà prêt à l'emploi. Quant à Subsonic, les login administrateurs sont *admin* pour le nom d'utilisateur et le mot de passe ; et concernant Emby il y a un *setup wizard* qui vous attend.

Je n'en ai pas parlé, mais les plus gros consommateurs de torrents aiment particulièrement automatiser leur *workflow* : Sickrage, Jackett... xataz a travaillé sur des images du genre, et chacune dispose d'un README qui vous aidera à les utiliser correctement : https://github.com/xataz/dockerfiles.

C'est tout pour maintenant. Ce n'est pas vraiment un tutoriel complet, mais par la même occasion j'ai pu montrer qu'on pouvait effectivement faire des choses sympathiques avec Docker. Si quelque chose n'est pas assez clair, merci de m'en faire part par le biais des commentaires histoire que je puisse améliorer l'article.
