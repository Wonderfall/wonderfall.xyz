Je n'ai peut-être pas la science infuse, mais la vision de `Dockerfiles` sans queue ni tête me trouble particulièrement. Avec Docker, je me considère encore comme un simple débutant. Pourtant, qu'est-ce qu'il y a de mal à s'appliquer dès ses débuts dans ce que l'on fait, pour éviter les mauvaises habitudes ? Et pourquoi ne pas appliquer quelques astuces ? J'ai souhaité en rassembler une partie dans cet article !

Le but de cet article n'est pas de savoir ce qu'est un `Dockerfile`, ni même comment on peut obtenir une image à partir de celui-ci. La documentation de Docker est suffisamment complète pour [cela](https://docs.docker.com/engine/reference/builder/). De même, un [guide des bonnes pratiques](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/), que nous allons partiellement revoir, est disponible. 

Le but est de fournir est un cheminement que je trouve à mon sens efficace pour **optimiser un Dockerfile**, à travers plusieurs aspects et astuces.

#### La légèreté/simplicité

Au sens du poids de l'image finale. Un `Dockerfile` se basera très probablement sur une image déjà existante, et ceci se fait avec la directive `FROM` obligatoirement au début du Dockerfile, de cette manière :

```language-docker
FROM image
```

Dans son comportement par défaut, Docker cherchera à télécharger l'image en question depuis le [Docker Hub](https://hub.docker.com/).  Or nombreux sont ceux qui se plaignent de devoir télécharger des images entières d'`ubuntu`, de `debian` et de voir leur espace disque saturer très rapidement. Avons-nous vraiment besoin d'`ubuntu` entièrement ? `ubuntu:trusty`, c'est déjà 188 Mo, pensez-y. Tant qu'à faire, préférez très largement `debian:jessie` qui fait 63 Mo de moins, et qui est par ailleurs une distribution de meilleure qualité aux dépôts très fournis.

**Oui, mais** : comment peut-on faire *encore* mieux ? On serait tenté de prendre `busybox`, après tout, pourquoi pas. C'est une image minimale et fonctionnelle, mais qui est loin de faciliter la vie. **C'est pour cela qu'Alpine Linux existe.**

![](https://pix.schrodinger.io/krFwiSGq/pPZs3Ky0.png)

Alpine Linux est une distribution GNU/Linux basée sur [musl](http://www.musl-libc.org/) et [Busybox](https://busybox.net/), qui se veut **légère, simple et sécurisée**. [Ses dépôts versionnés](https://pkgs.alpinelinux.org/packages) constituent son avantage majeur, qui fait d'elle la distribution minimale accessible et utilisable sans effort ; comme une autre distribution classique en fait, sauf que tout est dans le poids : seulement 5 Mo ! J'espère que vous m'aurez compris : si vous en êtes dans la possibilité (95% des cas, estimé très grossièrement...), **utilisez Alpine Linux**. Tout n'est pas rose, évidemment, des paquets sont absents, et je me suis souvent arraché les cheveux pour devoir compiler un truc et finalement me rabattre sur une bonne vieille Debian. Mais dans la majorité des cas, Alpine Linux a su correspondre à mes besoins. Les économies d'espace sont impressionnantes.

- `ubuntu:latest` : 188MB
- `debian:latest` : 125MB
- `opensuse:latest` : 98MB
- `busybox:latest` : **1MB**
- `alpine:latest` : **5MB**

**Note :** musl, une alternative de *libc* face à glibc, peut poser quelques problèmes pour utiliser certaines applications comme Mono ou OpenJDK. Pour contourner ça, je procède ainsi : 

```language-docker
ARG GLIBC_VER=2.23-r1
ARG GLIBC_URL=https://github.com/andyshinn/alpine-pkg-glibc/releases/download/$GLIBC_VER

RUN wget $GLIBC_URL/glibc-$GLIBC_VER.apk -O /tmp/glibc-$GLIBC_VER.apk \
 && wget $GLIBC_URL/glibc-bin-$GLIBC_VER.apk -O /tmp/glibc-bin-$GLIBC_VER.apk \
 && wget $GLIBC_URL/glibc-i18n-$GLIBC_VER.apk -O /tmp/glibc-i18n-$GLIBC_VER.apk \
 && wget https://www.archlinux.org/packages/extra/x86_64/mono/download/ -O /tmp/mono.pkg.tar.xz \
 && apk add --allow-untrusted \
    /tmp/glibc-$GLIBC_VER.apk \
    /tmp/glibc-bin-$GLIBC_VER.apk \
    /tmp/glibc-i18n-$GLIBC_VER.apk
```

À titre d'anecdote, Docker a embauché le créateur d'Alpine Linux, *Natanael Copa*, dans le but de [migrer toutes les images officielles](https://www.brianchristner.io/docker-is-moving-to-alpine-linux/), ou du moins le maximum, vers Alpine Linux. Docker prend désormais ce sujet au sérieux, autrefois source de pics faciles.

À l'utilisation d'Alpine Linux ou de Debian, il est systématiquement appréciable de nettoyer le cache du gestionnaire de paquets (généralement **tous les fichiers statiques** qui vous seront inutiles). Pour Alpine Linux, on se contente de :

```language-docker
RUN rm -f /var/cache/apk/*
```

Concernant Debian, j'ai toujours trouvé qu'`APT` faisait très usine à gaz, bien qu'il soit complet et intelligent (pas toujours...) pour résoudre des situations épineuses. Sauf que là, on se retrouve parfois avec des résidus incluant caches et paquets inutiles au fonctionnement de l'image finale. Pour cela, je recommande de dresser une liste **précise** des paquets nécessaires au fonctionnement d'une application. Ainsi, on peut les installer en se passant explicitement des paquets recommandés :

```language-docker
RUN apt-get install -y --no-install-recommends \
    truc \
    bidule \
    machin
```

Quand on n'a plus besoin d'un paquet, par exemple utilisé lors d'une compilation, on doit utiliser `apt-get purge` au lieu d'un simple `remove`. Enfin, pour être sûr de l'absence de paquets résiduels et de cache, on peut lancer ceci à la fin d'une instruction `RUN` dans laquelle a été lancé précédemment `apt-get install` :

```language-docker
RUN ... \
 && apt-get autoremove -y --purge \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*
```

De manière générale il faut songer à avoir une image finale propre. Si on utilise `pip`, on n'hésitera pas à passer le flag `--no-cache`. Quand on utilise `npm`, on peut se permettre de *passer un coup* avec `npm cache clean`.

Sommes-nous sauvés pour autant ? Pas vraiment, mais je vais en venir au fait. Vous devez savoir, si ce n'est pas déjà le cas, que la construction d'une image se fait grâce à l'union de *couches*, qu'on appelle *layers*. Cette *superposition de layers* qui constitue l'image finale se fait de haut en bas à partir des instructions du Dockerfile. 

**Déduction** : il est primordial de savoir qu'un fichier créé dans une instruction `RUN` ne peut être réellement supprimé que dans cette même instruction. Par exemple :

```language-docker
RUN wget -q https://ghost.org/zip/ghost-$VERSION.zip -P /tmp \
 && unzip -q /tmp/ghost-$VERSION.zip -d /ghost

RUN rm -rf /tmp/*
```

C'est inutile. Voici ce qu'il faut faire :

```language-docker
RUN wget -q https://ghost.org/zip/ghost-$VERSION.zip -P /tmp \
 && unzip -q /tmp/ghost-$VERSION.zip -d /ghost \
 && rm -rf /tmp/*
```

Que se passe-t-il dans le premier cas ? Pourquoi le fichier est absent de mon système de fichiers, finalement ? En fait, si le fichier n'est pas réellement supprimé, il est simplement **déréférencé** du `fs` final. Je le vois encore trop souvent et cela valait donc la remarque. Néanmoins, une fois que ce principe est assimilé par le concepteur, aucun problème du genre ne devrait subsister dans un Dockerfile.

Au passage, il ne faut installer que le strict nécessaire au bon fonctionnement de l'application. Rien de plus, rien de moins.

#### La flexibilité

Dans le cas particulier où vous souhaitez que votre image soit utilisable par tous, mais peut-être pas seulement, il est impératif de penser à la flexibilité. Les directives généralement utilisées dans ce contexte seront `ENV` et `ARG`. La première permet de passer des variables d'environnement à l'image, à partir du moment où la directive est utilisée lors du *build-time* jusqu'à l'utilisation de l'image finale dans un conteneur. La seconde permet de le faire également, mais seulement (et uniquement !) durant le *build-time*. 

Contrairement à `ARG`, `ENV` se destine donc à un usage persistent. L'utilisation d'`ENV` se déroule généralement à l'aide d'un script lancé avec la directive `CMD` (instruction qui se lance avec le *PID 1*). Une utilisation classique est **le choix du GID/UID** par l'utilisateur. Monter des volumes dans un conteneur est très fréquent, et l'utilisateur souhaite peut-être que ses conteneurs soient harmonisés, sans devoir par exemple utiliser `chown` à chaque fois avant d'en lancer un. Et parfois, des conteneurs seraient incompatibles entre eux s'ils utilisent des volumes en commun...

Concernant `ARG`, son utilité se révèle à mon avis dans le versioning des applications téléchargées. Exemple :

```language-docker
ARG VERSION=0.7.6
RUN wget -q https://ghost.org/zip/ghost-$VERSION.zip -P /tmp
```

Une valeur par défaut peut être assignée à l'aide de l'opérateur classique `=`. Ainsi, sans mettre à jour le Dockerfile, on peut facilement construire une image avec une version antérieure/ultérieure :

```language
docker build --build-arg VERSION=0.7.5 . 
```

Bien sûr, `ARG` ne se limite pas qu'à ça. On peut l'utiliser pour bien d'autres choses, comme pour offrir une gestion souple du nombre de cœurs à utiliser si des compilations sont à faire avec `make`.

```language-docker
ARG BUILD_CORES
RUN NB_CORES=${BUILD_CORES-`getconf _NPROCESSORS_CONF`}
RUN make -j $NB_CORES ...
```

La flexibilité peut aussi se manifester à travers l'utilisation de la directive `ENTRYPOINT`. Cette dernière permet de définir en quelque sorte une commande *de base* à partir de laquelle on peut adjoindre des flags, passer des paramètres... Cette adjonction peut être définie *par défaut* avec la directive `CMD`. Exemple tout bête et inutile :

```language-docker
ENTRYPOINT ["node"]
CMD ["-v"]
```

**Note** : ~~À ce jour, Docker Hub ne supporte pas l'utilisation des `ARG`. Une trop vieille version du Docker Engine est utilisée.~~ Maintenant vous pouvez utiliser les nouvelles features introduites dans la 1.9 pour vos automated builds !

Pour faire profiter ses travaux à tout le monde, vos Dockerfiles devront faire preuve de souplesse et de flexibilité. Je dépeins seulement des cas communs, tout dépend de l'application isolée ; mais comme le dit cette règle générale :

>Plus de layers pour plus de flexibilité, moins de layers pour plus de légèreté et moins de complexité.

C'est également une histoire de compromis !

#### La lisibilité

Pourquoi faire attention à la lisibilité d'un Dockerfile ?

- Plus facile à maintenir.
- Plus facile à comprendre.
- Plus facile à améliorer, favorisant prise de recul.

De manière générale, la lisibilité d'un Dockerfile est donc la bienvenue. Pensez tout d'abord à respecter une certaine harmonie : ne sautez pas des lignes inutilement, indentez correctement, et évitez les instructions longues et complexes quand elles peuvent être simples et courtes. Par exemple, il est bien plus lisible d'ériger une liste de paquets sous cette forme :

```language-docker```
RUN apk -U add \
    build-base \
    ca-certificates \
    bash \
    grep
```

Bien sûr, c'est d'autant plus un avantage quand la liste se rallonge. Libre à vous de trier par groupe de paquets en rapport, voire de façon alphabétique...

Dans une instruction `RUN`, on est souvent amené à faire une succession de commandes collées les unes aux autres avec `&&`. Personnellement, je préfère mettre les `\` en fin de ligne et les `&&` en début de ligne, avec un espace avant. [Voici un exemple.](https://lab.schrodinger.io/Wonderfall/Dockerfiles/src/master/ghost/0.7/Dockerfile)

La lisibilité d'un Dockerfile passe également par l'utilisation de directives qui ne semblent pas forcément utiles pour le bon fonctionnement de l'image et qui n’altèreraient pas son fonctionnement. Veillez à utiliser `EXPOSE` pour définir les ports utilisés par le conteneur, et à utiliser `VOLUME` pour définir les emplacements dans lesquels le conteneur écrit et lit des données, et qui devront être présents sur l'hôte pour la persistance. Une autre directive sympathique est `LABEL` qui permet de fournir des informations accessibles avec un `docker inspect`.

> Don't be such a syntax nazi.

Meh. Un Dockerfile respecte une certaine syntaxe, et parce qu'il le fait, on devrait produire de la lisibilité comme avec n'importe quel code source. Il n'y a pas de règles spécifiques, mais dans l'idéal l'écriture d'un Dockerfile devrait être conventionnée. Un peu de rigidité ne fait ici pas de mal, bien au contraire.

#### La sécurité

C'est l'objectif prépondérant. La [sécurité](https://docs.docker.com/engine/security/security/) est un thème sensible au sein de Docker, si bien que cela mériterait un article à part entière. Mais comme on ne parle que des Dockerfiles, ça va être plus synthétisable.

Isoler une application dans un conteneur n'a strictement aucun intérêt si celle-ci tourne en *root*. Tout comme un *chroot*, il n'est pas impossible d'échapper d'un conteneur pour atteindre l'hôte (et exécuter du code malveillant, etc.). Bien qu'on puisse utiliser un LSM (Linux Security Module) pour éviter le pire, ce qui est fortement recommandé (mais pas l'objet de l'article), ou encore les namespaces de Linux, une application **ne doit jamais tourner avec les droits root**. La directive `USER` permet de lancer la commande définie par `CMD` au conteneur avec un utilisateur non-root. Si `USER` ne peut être utilisé directement, utilisez `gosu` [(voir projet)](https://github.com/tianon/gosu). Illustration :

```language
gosu $GID:$UID commande
```

**EDIT (07/05/16)** : Pour cet usage, vous pouvez préférer [su-exec](https://github.com/ncopa/su-exec) qui est plus léger et qui est aussi dans les dépôts d'Alpine. *[`su-exec`] does more or less exactly the same thing as `gosu` but it is only 10kb instead of 1.8MB.*

De façon générale, **les pleins privilèges doivent être abandonnés dès que possible**. Naturellement, les **ports inférieurs à 1024 deviennent inutilisables**, mais ce n'est pas un problème puisqu'on fait très aisément du *port binding* entre le conteneur et l'hôte. Un exemple est ce que je fais avec [mon image nginx](https://github.com/Wonderfall/dockerfiles/tree/master/nginx) qui tourne tous les processus de nginx, y compris le *master process*, sous des permissions régulières et en utilisant les ports 8000 et 4430.

L'image de base téléchargée avec la directive `FROM` doit très préférablement utiliser un *tag*. Par exemple, au lieu d'utiliser `alpine`, j'utilise `alpine:3.3`. Ainsi, je ne vais pas être surpris de voir mon Dockerfile inutilisable du jour au lendemain. Par ailleurs, **utilisez des images officielles ou de confiance** (les vôtres naturellement). Ne faites pas confiance aveuglément à ce que vous trouvez sur le Hub.

Quant aux applications téléchargées dans les instructions `RUN`, quelques recommandations :

- Vérifier l'intégrité avec le checksum (`md5sum`, `sha256sum`).
- Vérifier l'authenticité avec `gpg` (et des clés).
- Utiliser HTTPS et ne pas esquiver la vérification des certificats.
- Le choix des sources (officielles de préférence), bien sûr.
- Les manipulations du type `curl | sh` sont à prohiber.

Exemple avec le tarball d'ownCloud :

```language-docker
 && OWNCLOUD_TARBALL="owncloud-${OWNCLOUD_VERSION}.tar.bz2" \
 && wget -q https://download.owncloud.org/community/${OWNCLOUD_TARBALL} \
 && wget -q https://download.owncloud.org/community/${OWNCLOUD_TARBALL}.sha256 \
 && wget -q https://download.owncloud.org/community/${OWNCLOUD_TARBALL}.asc \
 && wget -q https://pecl.php.net/get/apcu-${APCU_VERSION}.tgz \
 && wget -q https://pecl.php.net/get/apcu_bc-${APCUBC_VERSION}.tgz \
 && echo "Verifying both integrity and authenticity of ${OWNCLOUD_TARBALL}..." \
 && CHECKSUM_STATE=$(echo -n $(sha256sum -c ${OWNCLOUD_TARBALL}.sha256) | tail -c 2) \
 && if [ "${CHECKSUM_STATE}" != "OK" ]; then echo "Warning! Checksum does not match!" && exit 1; fi \
 && gpg --recv-keys F6978A26 \
 && FINGERPRINT="$(LANG=C gpg --verify ${OWNCLOUD_TARBALL}.asc ${OWNCLOUD_TARBALL} 2>&1 \
  | sed -n "s#Primary key fingerprint: \(.*\)#\1#p")" \
 && if [ -z "${FINGERPRINT}" ]; then echo "Warning! Invalid GPG signature!" && exit 1; fi \
 && if [ "${FINGERPRINT}" != "${GPG_owncloud}" ]; then echo "Warning! Wrong GPG fingerprint!" && exit 1; fi \
 && echo "All seems good, now unpacking ${OWNCLOUD_TARBALL}..." \
```

#### Layering et caching

Pour optimiser les temps de build d'un Dockerfile, il faut savoir pleinement tirer parti du système de cache de Docker. En fait, le même principe est à comprendre : Docker, dont le système de fichiers repose sur [UnionFS](https://en.wikipedia.org/wiki/UnionFS), utilise des *layers* pour ses images. Si un *layer* déjà présent est **identique** au résultat attendu d'une instruction, Docker va le reprendre. Parfois, cela peut s'avérer embêtant dans certaines instructions `RUN`, mais un contrôle fin du caching est prévu dans une prochaine version de Docker. Pour le moment, il faut faire avec.

Pour cette raison, il est souvent convenu d'utiliser les directives `ADD` et `COPY` à la suite des instructions `RUN` quand cela est possible, ou plutôt dès que cela est possible. Si un fichier du projet venait à être modifié, Docker ne tiendra plus compte du cache de toutes les instructions qui suivent la directive `COPY`/`ADD` qui lui était associée.

Il faudrait garder à l'esprit que l'ordre des instructions n'est pas laissé au hasard dans un Dockerfile. Faire attention à cet ordre permet d'optimiser considérablement ses temps de build. Une instruction `RUN` doit aussi être correctement placée par rapport aux autres pour que l'image soit fonctionnelle et pour que le *layering* soit efficient.

Histoire d'insister (j'ai déjà relaté du principe qui va suivre), un **juste-milieu** est à trouver entre la complexité d'utiliser un tas de layers et la simplicité d'utiliser le moins de layers possibles. Séparer les instructions `RUN` **intelligemment** est le meilleur choix pour tirer profit des deux mondes.

Notez également que si votre image repose sur une autre image que vous avez créée, veillez à maintenir cette dernière en la buildant à nouveau de temps en temps. En effet les images `alpine` et `debian` sont régulièrement mises à jour, or celles-ci sont indispensables à une sécurité correcte.

#### Quid du multi-process ?
Une question épineuse qui revient sans cesse au sein de la jeune communauté de Docker. Si Docker recommande officiellement de se borner à un seul processus par conteneur, dans les faits, le contraire est très facilement réalisable, et on y est parfois contraint ne serait-ce pour apporter de la flexibilité à l'image.

Si on suivait la doctrine conceptuelle de Docker, il faudrait donc penser : 

> 1 conteneur => 1 processus 

Dans ce cas, l'utilisation d'un gestionnaire de conteneur tel que *docker-compose* n'est même pas à discuter. 

Personnellement, l'idée d'avoir plusieurs processus dans un même conteneur ne me dérange pas particulièrement, car sinon cela ajouterait une nouvelle couche de complexité en sacrifiant facilement la flexibilité. Pour utiliser plusieurs processus, j'utilise *supervisor* dans un conteneur : très simple à configurer, complet, et propre. Je suis donc le principe suivant : 

>1 conteneur => 1 service => N (>= 1) processus

Si vous êtes confronté à la question, libre à vous de suivre la voie *idéale* (qui n'est pas irréalisable) de Docker ou non.

#### Halte aux zombies !

[Cet article](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/) explique très bien le problème des processus *zombies*, si bien que que je ne vais pas prendre la peine de parler des causes ([ce problème](https://fr.wikipedia.org/wiki/Processus_zombie) peut concerner les systèmes UNIX en général) mais plutôt des effets. L'équipe Docker est visiblement au courant puisque ce problème sera normalement résolu dans la prochaine version 1.11 avec l'intégration de [containerd](https://github.com/docker/docker/issues/20361).

En attendant, on peut utiliser [tini](https://github.com/krallin/tini). [Disponible](https://pkgs.alpinelinux.org/package/community/x86_64/tini) dans Alpine Linux (dans *community edge* actuellement), c'est un *init* simple et léger qui a pour but de veiller à la bonne terminaison des processus avec un *forwarding* du signal d'arrêt (`docker stop` envoie un `SIGTERM` au PID 1) vers les processus enfants. De par sa légèreté (par rapport à systemd par exemple), il se rend parfait pour un conteneur. Si un init comme *tini* n'est pas utilisé, il en résulte au final dans un temps d'arrêt du conteneur **long** et dans des processus terminés **non-proprement** (`SIGKILL` envoyé par le kernel).

#### Conclusion

Pour obtenir des images légères, propres et sécurisées, une piste est d'avoir à l'esprit les aspects dont je vous ai parlés. Je vous propose cette piste, mais je ne vous l'impose pas !
