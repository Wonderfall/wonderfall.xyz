On continue donc la série des *"\* avec Docker"*, cette fois-ci avec la conteneurisation d'un relais [Tor](https://www.torproject.org/), un réseau mondial et décentralisé, mais surtout un des rares moyens permettant la navigation Web *a priori* anonyme (par exemple). On peut considérer Tor comme un service qui se découpe en deux parties : la partie client d'un côté avec les outils comme le [Tor Browser](https://www.torproject.org/download/download-easy.html.en), et la partie réseau qui vit principalement du bénévolat.

Quand Tor est utilisé, l'adresse IP que voit le serveur distant est celle d'un relais de sortie, qui peut se trouver n'importe où dans le monde. Mais résumer Tor à cela serait négliger sa puissante **capacité d'anonymisation** qui vise à réduire les traces, consistant en la circulation du trafic à travers une série de relais.

En effet, le réseau Tor est composé de serveurs, appelés **nœuds**, **routeurs** ou encore **relais** dans son contexte de fonctionnement, dont la liste est publique. On distingue plusieurs modes de configuration de relais au sein du réseau Tor :

- *Bridge*, relais *guard* cachés.
- *Middle* / *Guard*, relais intermédiaires.
- *Exit*, relais de sortie.

Pour comprendre ce qui différencie ces trois modes, il suffit de s'intéresser au fonctionnement de Tor. Pour commencer, voici des illustrations de l'[EFF](https://www.eff.org/) à ce sujet :

![](https://www.torproject.org/images/htw1.png)
![](https://www.torproject.org/images/htw2.png)
![](https://www.torproject.org/images/htw3.png)

Le circuit commence à partir d'un *guard* (point d'entée), puis termine au niveau d'un *exit* (point de sortie), en passant par au moins un relais intermédiaire.

> Client -> Guard (Bridge si censure) -> n>=1 Middle -> Exit -> Destination.

Ainsi, on peut dire que la qualité, la robustesse, et la rapidité du réseau reposent sur ces relais. Diversifier la présence des nœuds avec des machines saines permet par exemple de lutter contre les [sybil attacks](https://en.wikipedia.org/wiki/Sybil_attack) qui représentent la seule menace préoccupante à laquelle le réseau Tor peut envisager de faire face un jour. Ainsi, on peut dire que **plus de personnes installent de relais, mieux c'est !** De plus, les relais intermédiaires ne vous exposent pas à de gros risques, ce qui fait que vous pouvez très bien en héberger un chez vous si ça vous chante. Si vous en avez les moyens, je vous recommande de commencer avec ce mode.

Un relais *guard* (point d'entrée du circuit, donc en première position) est *privilégié* puisqu'il peut connaître l'IP source du client. Pour devenir un relais *guard* il suffit d'être un relais *middle* stable (avoir un bon uptime) avec une bonne bande passante. **Aucune configuration n'est nécessaire**, un relais *middle* se voit attribuer le flag *guard* automatiquement après un certain temps et sous ces conditions.

Cependant, on ne peut pas en dire autant des relais de sortie, qui constituent le point de sortie du circuit vers la destination finale. L'IP du relais de sortie est interprétée comme la source du trafic ; donc si des données illicites y transitent, cette IP sera perçue comme responsable en quelque sorte. Contrairement aux relais intermédiaires, le trafic qui circule par ces nœuds de sortie n'est pas sécurisé. C'est pourquoi **il est plus délicat d'héberger des relais de sortie**, bien que pénalement la législation européenne devrait en protéger les opérateurs. Ne vous lancez pas seul dans cette aventure si vous voulez éviter les mauvaises surprises.

Quant aux *bridges*, ce sont en fait des relais *guard* cachés qui ne sont pas listés publiquement comme appartenant au réseau Tor, car ils constituent un moyen de contourner le blocage et la censure, notamment en Chine. S'ils ne sont pas listés, ils ne peuvent pas être bloqués. Héberger un *bridge* ne vous expose à aucun gros risque. Ils sont d'ailleurs plus intéressants à mettre en place si vous disposez d'une faible bande passante.

Je vous invite à consulter [cette page](https://trac.torproject.org/projects/tor/wiki/doc/GoodBadISPs) avant d'héberger quoique ce soit, ça vous évitera de mauvaises surprises, sait-on jamais ce qui peut arriver avec certains ISPs. J'ai demandé par exemple à [Ikoula](https://www.ikoula.com/fr) ce qu'ils en pensaient :

> Mettre en place un Tor n'est en soit pas illégal. C'est son utilisation qui peut être incriminée. Nous ne l'interdisons pas, cependant si nous sommes contactés par les autorités nous le signalons au client. S'il ne répond pas, nous prenons des mesures.

À titre informatif, voici une proportion des relais en fonction de leurs *flags*, et son évolution d'Avril 2015 à Avril 2016 :

![](https://pix.schrodinger.io/LCZNnP7x/T93NwrD5.jpg)

Installer un relais Tor est très simple, puisqu'il suffit d'installer le paquet `tor` sur sa distribution GNU/Linux et de quelques lignes de configuration suivant le relais qu'on souhaite. J'ai jugé pertinent de poursuivre l'installation avec Docker (bien que ce soit déjà enfantin sans), et ainsi bénéficier d'un minimum d'isolation. On peut facilement lancer plusieurs conteneurs pour lancer plusieurs instances de Tor si souhaité (notamment parce que [Tor ne supporte pas le multi-threading](https://trac.torproject.org/projects/tor/ticket/1749)). [Le Dockerfile](https://github.com/Wonderfall/dockerfiles/blob/master/tor/Dockerfile) :

```language-docker,line-numbers
FROM alpine:3.3

ARG TOR_VERSION=0.2.7.6
ARG TOR_USER_ID=45553
ARG ARM_VERSION=1.4.5.0

ARG GPG_Mathewson="B35B F85B F194 89D0 4E28  C33C 2119 4EBB 1657 33EA"
ARG GPG_Johnson="6827 8CC5 DD2D 1E85 C4E4  5AD9 0445 B7AB 9ABB EEC6"

ENV TERM=xterm

RUN BUILD_DEPS=" \
    libevent-dev \
    openssl-dev \
    build-base \
    gnupg \
    ca-certificates" \
 && apk -U add \
    ${BUILD_DEPS} \
    python \
    libevent \
    openssl \
 && cd /tmp \
 && TOR_TARBALL="tor-${TOR_VERSION}.tar.gz" \
 && wget -q https://www.torproject.org/dist/${TOR_TARBALL} \
 && echo "Verifying ${TOR_TARBALL} using GPG..." \
 && wget -q https://www.torproject.org/dist/${TOR_TARBALL}.asc \
 && gpg --keyserver keys.gnupg.net --recv-keys 0x165733EA \
 && FINGERPRINT="$(LANG=C gpg --verify ${TOR_TARBALL}.asc ${TOR_TARBALL} 2>&1 \
  | sed -n "s#Primary key fingerprint: \(.*\)#\1#p")" \
 && if [ -z "${FINGERPRINT}" ]; then echo "Warning! Invalid GPG signature!" && exit 1; fi \
 && if [ "${FINGERPRINT}" != "${GPG_Mathewson}" ]; then echo "Warning! Wrong GPG fingerprint!" && exit 1; fi \
 && echo "All seems good, now unpacking ${TOR_TARBALL}..." \
 && tar xzf ${TOR_TARBALL} && cd tor-${TOR_VERSION} \
 && ./configure --disable-asciidoc && make && make install \
 && adduser -h /var/run/tor -D -s /sbin/nologin -u ${TOR_USER_ID} tor \
 && cd /tmp \
 && ARM_TARBALL="arm-${ARM_VERSION}.tar.bz2" \
 && wget -q https://www.atagar.com/arm/resources/static/${ARM_TARBALL}  \
 && echo "Verifying ${ARM_TARBALL}..." \
 && wget -q https://www.atagar.com/arm/resources/static/${ARM_TARBALL}.asc \
 && gpg --keyserver pgp.mit.edu --recv-keys 0x9ABBEEC6 \
 && FINGERPRINT="$(LANG=C gpg --verify ${ARM_TARBALL}.asc ${ARM_TARBALL} 2>&1 \
  | sed -n "s#Primary key fingerprint: \(.*\)#\1#p")" \
 && if [ -z "${FINGERPRINT}" ]; then echo "Warning! Invalid GPG signature!" && exit 1; fi \
 && if [ "${FINGERPRINT}" != "${GPG_Johnson}" ]; then echo "Warning! Wrong GPG fingerprint!" && exit 1; fi \
 && echo "All seems good, now unpacking ${ARM_TARBALL}..." \
 && tar xjf /tmp/${ARM_TARBALL} && cd arm && ./install \
 && apk del ${BUILD_DEPS} \
 && rm -rf /var/cache/apk/* /tmp/*

VOLUME /usr/local/etc/tor /tordata
EXPOSE 9001 9030
USER tor

ENTRYPOINT [ "tor" ]
```

On utilise Alpine Linux pour partir d'une base légère. Ici, nous compilerons `tor` depuis les sources, après vérification de l'authenticité du tarball que nous avons téléchargé. Les ports `9001` et `9030` seront exposés, et les volumes que nous utiliserons seront `/usr/local/etc/tor` et `/tordata`. Le conteneur ne peut plus utiliser *root* car nous avons précisé l'utilisateur *tor* avec la directive `USER`. Enfin nous n'avons pas défini de commande avec `CMD` mais un simple point d'entrée pour laisser un peu plus de flexibilité dans le choix des paramètres de lancement.

L'image possède une version d'`arm` (*Tor-ARM*) installée. [Tor-ARM](https://www.torproject.org/projects/arm.html.en) est un soft qui vous permettra de faire du *real-time monitoring* de votre relais, par exemple comme le fait `top`. Si vous voulez l'utiliser, il faudra ajouter quelques trucs dans votre configuration.

Si vous n'avez pas envie de construire l'image manuellement, sachez que j'en propose une *automated build* ici : [wonderfall/tor](https://hub.docker.com/r/wonderfall/tor/). On peut télécharger l'image pour commencer :

```language
$ docker pull wonderfall/tor
```

Cela dit, **je ne recommande pas de procéder ainsi**. Vous ferez plus confiance à une image que vous avez construite vous-même qu'une autre construite par un tiers. Pour construire l'image, il suffit simplement de faire :

```language
$ docker build -t wonderfall/tor github.com/wonderfall/dockerfiles.git#master:tor
```

Les volumes seront montés dans `/home/docker/tor/data` pour `/tordata` et `/home/docker/tor/config` pour `/usr/local/etc/tor`. Libre à vous de procéder autrement, adaptez seulement les chemins. On commence donc par créer les dossiers dont nous avons besoin :

```language-bash
$ mkdir -p /home/docker/tor/{data,config}
```

Puis on se rend dans `/home/docker/tor/config` pour créer nos fichiers de configuration. N'oubliez pas de changer ceci :

- `${NICKNAME}` : *pseudonyme* de votre relais, il sera listé publique avec ce nom et vous pourrez le retrouver ainsi sans utiliser son *fingerprint*.
- `${CONTACT_NAME}` : votre pseudonyme.
- `${CONTACT_EMAIL}` : votre e-mail de contact, ne rentrez pas n'importe quoi non plus car vous pouvez recevoir des informations utiles.
- `${BANDWITH_RATE}` : bande passante à utiliser en Mbps.
- `${BANDWITH_BURST}` : dépassement de la BP en Mbps.
- `${ADDRESS}` : IP ou FQDN du serveur.

Vous acceptez de toute façon que **ces données soient publiques**. Évitez de mettre votre e-mail personnel si vous ne voulez pas risquer les spambots. Vous pouvez aussi fournir l'empreinte de votre clé publique PGP (dans `ContactInfo`).

`/home/docker/tor/config/torrc.bridge` :
```language
ORPort 9001
DirPort 9030
Address ${ADDRESS}
BandwidthRate ${BANDWITH_RATE}
BandwidthBurst ${BANDWITH_BURST}
Nickname ${NICKNAME}
ContactInfo ${CONTACT_NAME} <${CONTACT_EMAIL}>
BridgeRelay 1
```

`/home/docker/tor/config/torrc.middle` :
```language
ORPort 9001
DirPort 9030
Address ${ADDRESS}
BandwidthRate ${BANDWITH_RATE}
BandwidthBurst ${BANDWITH_BURST}
Nickname ${NICKNAME}
ContactInfo ${CONTACT_NAME} <${CONTACT_EMAIL}>
ExitPolicy reject *:*
```

`/home/docker/tor/config/torrc.exit` :
```language
ORPort 9001
DirPort 9030
Address ${ADDRESS}
BandwidthRate ${BANDWITH_RATE}
BandwidthBurst ${BANDWITH_BURST}
Nickname ${NICKNAME}
ContactInfo ${CONTACT_NAME} <${CONTACT_EMAIL}>

# Reduced exit policy from
# https://trac.torproject.org/projects/tor/wiki/doc/ReducedExitPolicy
ExitPolicy accept *:20-23     # FTP, SSH, telnet
ExitPolicy accept *:43        # WHOIS
ExitPolicy accept *:53        # DNS
ExitPolicy accept *:79-81     # finger, HTTP
ExitPolicy accept *:88        # kerberos
ExitPolicy accept *:110       # POP3
ExitPolicy accept *:143       # IMAP
ExitPolicy accept *:194       # IRC
ExitPolicy accept *:220       # IMAP3
ExitPolicy accept *:389       # LDAP
ExitPolicy accept *:443       # HTTPS
ExitPolicy accept *:464       # kpasswd
ExitPolicy accept *:465       # URD for SSM (more often: an alternative SUBMISSION port, see 587)
ExitPolicy accept *:531       # IRC/AIM
ExitPolicy accept *:543-544   # Kerberos
ExitPolicy accept *:554       # RTSP
ExitPolicy accept *:563       # NNTP over SSL
ExitPolicy accept *:587       # SUBMISSION (authenticated clients [MUA's like Thunderbird] send mail over STARTTLS SMTP here)
ExitPolicy accept *:636       # LDAP over SSL
ExitPolicy accept *:706       # SILC
ExitPolicy accept *:749       # kerberos
ExitPolicy accept *:873       # rsync
ExitPolicy accept *:902-904   # VMware
ExitPolicy accept *:981       # Remote HTTPS management for firewall
ExitPolicy accept *:989-995   # FTP over SSL, Netnews Administration System, telnets, IMAP over SSL, ircs, POP3 over SSL
ExitPolicy accept *:1194      # OpenVPN
ExitPolicy accept *:1220      # QT Server Admin
ExitPolicy accept *:1293      # PKT-KRB-IPSec
ExitPolicy accept *:1500      # VLSI License Manager
ExitPolicy accept *:1533      # Sametime
ExitPolicy accept *:1677      # GroupWise
ExitPolicy accept *:1723      # PPTP
ExitPolicy accept *:1755      # RTSP
ExitPolicy accept *:1863      # MSNP
ExitPolicy accept *:2082      # Infowave Mobility Server
ExitPolicy accept *:2083      # Secure Radius Service (radsec)
ExitPolicy accept *:2086-2087 # GNUnet, ELI
ExitPolicy accept *:2095-2096 # NBX
ExitPolicy accept *:2102-2104 # Zephyr
ExitPolicy accept *:3128      # SQUID
ExitPolicy accept *:3389      # MS WBT
ExitPolicy accept *:3690      # SVN
ExitPolicy accept *:4321      # RWHOIS
ExitPolicy accept *:4643      # Virtuozzo
ExitPolicy accept *:5050      # MMCC
ExitPolicy accept *:5190      # ICQ
ExitPolicy accept *:5222-5223 # XMPP, XMPP over SSL
ExitPolicy accept *:5228      # Android Market
ExitPolicy accept *:5900      # VNC
ExitPolicy accept *:6660-6669 # IRC
ExitPolicy accept *:6679      # IRC SSL
ExitPolicy accept *:6697      # IRC SSL
ExitPolicy accept *:8000      # iRDMI
ExitPolicy accept *:8008      # HTTP alternate
ExitPolicy accept *:8074      # Gadu-Gadu
ExitPolicy accept *:8080      # HTTP Proxies
ExitPolicy accept *:8082      # HTTPS Electrum Bitcoin port
ExitPolicy accept *:8087-8088 # Simplify Media SPP Protocol, Radan HTTP
ExitPolicy accept *:8332-8333 # Bitcoin
ExitPolicy accept *:8443      # PCsync HTTPS
ExitPolicy accept *:8888      # HTTP Proxies, NewsEDGE
ExitPolicy accept *:9418      # git
ExitPolicy accept *:9999      # distinct
ExitPolicy accept *:10000     # Network Data Management Protocol
ExitPolicy accept *:11371     # OpenPGP hkp (http keyserver protocol)
ExitPolicy accept *:19294     # Google Voice TCP
ExitPolicy accept *:19638     # Ensim control panel
ExitPolicy accept *:50002     # Electrum Bitcoin SSL
ExitPolicy accept *:64738     # Mumble
ExitPolicy reject *:*
```

Ces configurations sont des exemples fonctionnels et minimaux, vous pouvez les **modifier** comme bon vous semble (vous devriez même). Par exemple, pourquoi ne pas définir un plafond de la consommation de bande passante avec les directives `AccountingMax` et `AccountingStart` ? Pratique si vous avez un quota mensuel à respecter. Veillez à bien configurer les paramètres de la bande passante puisque vous ne voulez peut-être pas rendre votre serveur inutilisable pour d'autres choses, d'autant plus que la qualité du réseau Tor peut en dépendre (voir [ici](https://blog.torproject.org/running-exit-node) au point 7.).

Pour utiliser `arm`, il faudra ajouter la directive `ControlPort` renseignée avec un port (> 1000) choisi arbitrairement. Si vous ne liez pas ce port à l'hôte, aucune raison de vous inquiéter. Sinon, il aurait fallu renseigner la directive `HashedControlPassword` avec un mot de passe hashé obtenu avec `tor --hash-password password`. Je ne vais pas m'étaler davantage, la documentation est faite pour cela : https://www.torproject.org/docs/tor-manual.html.en

Ensuite on va attribuer les permissions spécifiques à l'utilisateur `tor` créé dans l'image au dossier `data`. Si vous avez simplement copié/collé le Dockerfile, ou téléchargé l'image, ce sera `45553:45553`. Sinon sachez que vous pouvez modifier l'ID de l'utilisateur `tor` avec l'argument `TOR_USER_ID` renseigné lors de la construction de l'image.

```language-bash
$ chown 45553:45553 /home/docker/tor/data
```

En théorie il n'y a plus qu'à lancer le conteneur, comme ceci par exemple dans le cas d'un relais intermédiaire :

```language-bash
$ docker run -d \
    -v /etc/localtime:/etc/localtime \
    -v /home/docker/tor/data:/tordata \
    -v /home/docker/tor/config:/usr/local/etc/tor \
    --restart always \
    -p 9001:9001 \
    -p 9030:9030 \
    --name tor-relay \
    wonderfall/tor -f /usr/local/etc/tor/torrc.middle
```

Il est aussi envisageable d'utiliser `docker-compose`. Si vous utilisez la version 2 pour votre `docker-compose.yml`, vous pouvez vous faire plaisir en ayant directement accès aux `ARG`, variables uniquement utilisées lors de la construction de l'image.

```language-yaml
# --- V1 ---
torrelay:
# image: wonderfall/tor
  build: /chemin/vers/dockerfile/
  container_name: tor-relay
  ports:
    - "9001:9001"
    - "9030:9030"
  volumes:
    - /etc/localtime:/etc/localtime
    - /home/docker/tor/data:/tordata
    - /home/docker/tor/config:/usr/local/etc/tor
  command: -f /usr/local/etc/tor/torrc.middle

# --- V2 ---
torrelay:
# network_mode: "bridge"
  build:
    context: /chemin/vers/dockerfile/
    dockerfile: Dockerfile
    args:
      - TOR_VERSION=0.2.7.6
      - TOR_USER_ID=45553
      - ARM_VERSION=1.4.5.0
  image: wonderfall/tor
  container_name: tor-relay
  ports:
    - "9001:9001"
    - "9030:9030"
  volumes:
    - /etc/localtime:/etc/localtime
    - /home/docker/tor/data:/tordata
    - /home/docker/tor/config:/usr/local/etc/tor
  command: -f /usr/local/etc/tor/torrc.middle
```

J'en profite justement pour insister sur l'importance de **maintenir une version de Tor à jour**. Je mets à jour [le Dockerfile de mon git](https://raw.githubusercontent.com/Wonderfall/dockerfiles/master/tor/Dockerfile) aussi vite que possible, mais si je ne le fais pas, ne m'attendez pas. Vous pouvez changer la ligne suivante dans le Dockerfile :

```language-docker
ARG TOR_VERSION=version
```
Ou utiliser la seconde version du format d'un `docker-compose.yml` comme je l'ai montré plus haut avec un exemple, ou encore construire l'image ainsi :

```language
$ docker build --build-arg TOR_VERSION=version .  
```

Maintenant que c'est dit, on vérifie que tout fonctionne bien après avoir attendu quelques secondes le temps que le conteneur se connecte au réseau Tor :

```language-bash
$ docker logs tor-relay
```

Normalement les messages sont assez explicites pour que vous puissiez déterminer si le relais fonctionne ou non. Pour utiliser `arm` :

```language-bash
$ docker exec -ti tor-relay arm -i $CONTROLPORT
```

![](https://pix.schrodinger.io/j5hYA0aD/mb9hHBj0.jpg)

Après quelques heures de propagation, vous pouvez vérifier que votre serveur est bien reconnu comme participant au réseau Tor sur [Globe](https://globe.torproject.org/) ou [Atlas](https://atlas.torproject.org/). Vous serez gratifié pour votre stabilité si vous laissez tourner votre conteneur un certain temps. Pas de panique si votre relais est sous-utilisé dans un premier temps, celui-ci va suivre plusieurs étapes (dans un cycle de maturation) décrites [ici](https://blog.torproject.org/blog/lifecycle-of-a-new-relay) avant que le réseau ne lui fasse pleinement confiance.

Remerciements :

- à Jess et [son article](https://blog.jessfraz.com/post/running-a-tor-relay-with-docker/) qui m'a fortement inspiré.
- à [Angristan](https://angristan.fr/) pour ses précisions.
- à [Aeris](https://imirhil.fr/) pour son échange intéressant sur Twitter.
