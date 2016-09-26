Hey everyone! Pendant ce week-end orageux, je me suis lancé un petit défi (inutile j'en conviens) : compiler nginx avec BoringSSL, un *fork* d'[OpenSSL](https://www.openssl.org/) par Google qui est notamment un toolkit SSL/TLS. Commençons donc par expliquer en quoi consiste le projet [BoringSSL](https://boringssl.googlesource.com/boringssl/), et qu'est-ce qui le différencie par rapport à OpenSSL et à [LibreSSL](http://www.libressl.org/), un autre fork qui je le rappelle a été lancé par l'équipe derrière [OpenBSD](http://www.openbsd.org/) à la suite de la déferlante (mais pas si surprenante) qu'a été [Heartbleed](https://en.wikipedia.org/wiki/Heartbleed) en 2014. On peut considérer que BoringSSL est né de cette même impulsion qui a motivé la création de LibreSSL, mais en réalité, la décision a été motivée par la volonté de Google à simplifier les choses de son côté. Google utilisait depuis un long moment une version d'OpenSSL avec une accumulation de patchs de plus en plus conséquente, utilisée par exemple dans AOSP ou Chromium.

Il semblait alors évident de partir sur une base propre, et c'est ce qui a vraiment été fait. BoringSSL n'a pas été "copié" depuis OpenSSL directement ; le travail a commencé à partir d'un dossier vide, et fonction par fonction le code d'OpenSSL a été radicalement amélioré (documentations, formatages, nettoyages et suppressions). De nombreux algorithmes et fonctionnalités sont aux abonnés absents : Camellia, IDEA, MD2, PKCS#7, RC5, la compression, l'extension heartbeat... Alors que la base de départ (une version d'OpenSSL 1.0.2) contenait 468 000 lignes de codes, BoringSSL en contient plus de deux fois moins (~200 000). De nombreuses fonctions d'OpenSSL jugées désuettes ont été dépréciées, pouvant impliquer parfois quelques problèmes de compatibilité, mais c'est cette liberté que Google veut pouvoir se permettre, contrairement à LibreSSL qui se concentre principalement sur la qualité du code plutôt que de chambouler toute l'ABI. Notons également que BoringSSL supporte la version IETF du cipher [ChaCha20-Poly1305](https://tools.ietf.org/html/rfc7539), une alternative à AES plus rapide sur les appareils ne disposant pas des instructions [AES-NI](https://en.wikipedia.org/wiki/AES_instruction_set), les courbes elliptiques [X25519 et ed25519](https://cr.yp.to/ecdh.html), et une implémentation de [TLS 1.3](https://tlswg.github.io/tls13-spec/) est dores et déjà en cours de développement.

Je ne saurais pas vraiment donner un intérêt à l'utilisation de BoringSSL avec nginx ; bien que BoringSSL soit certainement plus propre qu'OpenSSL, LibreSSL a le mérite d'être pouvoir utilisé très facilement en apportant a priori les mêmes bénéfices. On va également procéder à une compilation statique, mais quelques étapes intermédiaires sont à considérer en plus. Je fais cet article principalement pour cela, puisque les recherches sur Google ne donnent pas vraiment grand chose... **Cet article condense toutes les astuces nécessaires au jour de son écriture pour obtenir une version fonctionnelle.** Par ailleurs, je vais partir du principe que vous savez déjà compiler nginx. Il y a des centaines de tutoriels sur ce sujet, il faudra juste leur adjoindre les quelques étapes qui vont suivre.

>**Docker Way :** Pour les amoureux de Docker (comme moi), je propose une image [wonderfall/boring-nginx](https://hub.docker.com/r/wonderfall/boring-nginx/) dont l'utilisation est documentée sur Docker Hub et [Github](https://github.com/Wonderfall/dockerfiles/tree/master/boring-nginx).

Pour commencer, on va s'occuper de BoringSSL. Il faut considérer deux dépendances : **go** (lang) et **cmake**. On va construire les bibliothèques, et ensuite aménager un dossier `.openssl` pour que la compilation de nginx avec BoringSSL puisse se faire.

```language-bash
cd /tmp
git clone https://boringssl.googlesource.com/boringssl
cd boringssl
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j $(getconf _NPROCESSORS_CONF)
cd ..
include/openssl/crypto.h
mkdir -p .openssl/lib/
cd .openssl
ln -s ../include
cd ..
cp build/crypto/libcrypto.a build/ssl/libssl.a .openssl/lib
```

De retour vers nos sources d'nginx, il va tout d'abord falloir appliquer quelques modifications. Dans le cas contraire, il y a aura des erreurs aboutissant à l'arrêt de la compilation. En effet, BoringSSL a modifié en profondeur certaines fonctions, et on rencontre notamment une incompatibilité de types dans les paramètres de `SSL_set_tlsext_host_name()`. Certains identifiants sont obsolètes avec BoringSSL et nous allons en conséquence utiliser des préprocesseurs, sans quoi la compilation plantera. En plus, de nouveaux problèmes risquent d'apparaître plus tard... C'est pour cela que je me propose pour maintenir [ce patch](https://raw.githubusercontent.com/Wonderfall/dockerfiles/master/boring-nginx/boring.patch) qui, je pense, vous facilitera la tâche. Si pour une raison je venais à ne plus maintenir à jour ce patch, vous pouvez également suivre le travail de [ajhaydock/BoringNginx](https://github.com/ajhaydock/BoringNginx).

```language-bash
cd /tmp
wget https://raw.githubusercontent.com/Wonderfall/dockerfiles/master/boring-nginx/boring.patch
cd /tmp/nginx-${VERSION}
patch -p1 < /tmp/boring.patch
./configure \
   --with-cc-opt="-I ../boringssl/.openssl/include/" \
#  --with-cc-opt="-g -O3 -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=2 -fPIE -fstack-protector-strong -Wformat -Werror=format-security -I ../boringssl/.openssl/include/" \
   --with-ld-opt="-L ../boringssl/.openssl/lib"
#  --with-ld-opt="-Wl,-Bsymbolic-functions -Wl,-z,relro -L ../boringssl/.openssl/lib" \
make -j $(getconf _NPROCESSORS_CONF)
make install
```

Maintenant, on peut vérifier qu'on dispose comme prévu d'une version d'nginx statiquement compilée avec BoringSSL, avec la commande `nginx -V` qui devrait retourner un début de sortie similaire à ceci :

```language-bash
$ nginx -V
nginx version: nginx/1.11.1
built by gcc 5.3.0 (Alpine 5.3.0)
built with OpenSSL 1.0.2 (compatible; BoringSSL) (running with BoringSSL)
TLS SNI support enabled
configure arguments: ...
```

Avant d'utiliser cette version, quelques notes. Premièrement, **BoringSSL ne supporte pas l'OCSP stapling** et une explication est fournie [ici](https://www.imperialviolet.org/2014/04/19/revchecking.html). Afin d'éviter les warnings, retirez l'OCSP stapling de vos vhosts. Autre chose, une particularité intéressante chez BoringSSL est la possibilité de laisser le client choisir son cipher préféré dans un groupe de ciphers déterminé. Ceci est très utile puisque, par exemple, on ne veut pas forcément utiliser ChaCha20-Poly1305 sur du matériel qui supporte AES-NI. On perd en performances (cf. [les benchmarks](https://calomel.org/aesni_ssl_performance.html))... Mais d'un autre côté, on veut utiliser ChaCha20-Poly1305 pour les clients qui ne supportent pas AES-NI, parce que dans ce cas il est le plus rapide. On fait appel à une syntaxe particulière : le groupe est entre des brackets `[]`, et les ciphers sont "également préférables" quand ils sont séparés par des `|` (*ou*). Voici un exemple :

```language-nginx
ssl_ciphers [ECDHE-ECDSA-CHACHA20-POLY1305|ECDHE-RSA-CHACHA20-POLY1305|ECDHE-ECDSA-CHACHA20-POLY1305-D|ECDHE-RSA-CHACHA20-POLY1305-D|ECDHE-ECDSA-AES256-GCM-SHA384|ECDHE-RSA-AES256-GCM-SHA384|ECDHE-ECDSA-AES128-GCM-SHA256|ECDHE-RSA-AES128-GCM-SHA256]:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-ECDSA-AES256-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA;
```

Notez que j'ai ajouté la version *draft* de ChaCha20-Poly1305, puisque c'est une version plus répandue que la version IETF. Mais cette dernière sera celle qui sera utilisée par exemple dans [la prochaine version de Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=1247860). On peut très simplement vérifier cette suite avec le test de [Qualys SSL Labs](https://www.ssllabs.com/ssltest/) qui devrait donner ceci dans la partie *Cipher Suites* :

![](https://pix.schrodinger.io/3JVGujGV/HE64PE3e.png)

*This server prefers ChaCha20 suite with clients that don't have AES-NI [...]*. C'est donc bien ce qu'on voulait finalement. 

Retenez surtout que BoringSSL est développé *par Google pour Google*, donc vous ne bénéficierez pas par exemple de la possibilité introduite dans la mainline 1.11 de définir une liste de courbes elliptiques utilisables pour ECDH. BoringSSL a dégagé `SSL_CTX_set1_curves_list()` puisque Google n'en avait pas l'utilité, [m'a-t-on répondu sur Twitter](https://twitter.com/davidben__/status/739095518568194048). Donc ce n'est pas surprenant si même [un développeur de nginx](https://trac.nginx.org/nginx/ticket/993#comment:1) annonce que son projet ne suivra pas les moults changements que Google a apportés dans l'API de BoringSSL. Cette clause de BoringSSL résume assez bien ma mise en garde :

>Although BoringSSL is an open source project, it is not intended for general use, as OpenSSL is. We don't recommend that third parties depend upon it. Doing so is likely to be frustrating because there are no guarantees of API or ABI stability.
 
Je pense que c'est tout, n'hésitez pas à utiliser les commentaires si vous voulez plus de précisions par exemple. Si vous maîtrisez l'anglais, [cet article](https://www.imperialviolet.org/2015/10/17/boringssl.html) saura vous satisfaire davantage dans le cas où vous voulez en savoir plus sur BoringSSL. 
