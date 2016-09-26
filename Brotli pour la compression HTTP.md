À l'instar de [deflate](https://en.wikipedia.org/wiki/DEFLATE) ([gzip](https://fr.wikipedia.org/wiki/Gzip)), LZHAM et LZMA, [Brotli](https://en.wikipedia.org/wiki/Brotli) est un algorithme de compression sans perte de données, en [draft](http://www.ietf.org/id/draft-alakuijala-brotli-08.txt) à l'IEFT et déjà utilisé dans la norme [WOFF 2.0](https://fr.wikipedia.org/wiki/Web_Open_Font_Format_2) (pour *Web Open Font Format 2*). Brotli est entre autre lui-même basé sur une variante moderne de l'algorithme [LZ77](https://en.wikipedia.org/wiki/LZ77_and_LZ78) et sur le [codage de Huffman](https://en.wikipedia.org/wiki/Huffman_coding). Il a été développé par Jyrki Alakuijala et Zoltan Szabadka, tous deux sont ingénieurs software chez Google Zürich. Non pas sans hasard, Brotli porte apparemment le nom d'une viennoiserie suisse (ce n'est pas l'image de l'article, malheureusement je n'ai pas trouvé une image libre de droit pour illustrer avec un vrai brotli).

Pour les néophytes, un algorithme de compression de données permet de réduire la taille d'une certaine séquence de bits, dans l'intérêt de gagner de l'espace disque ou de **limiter le coût sur la bande passante**. Ici on entend bien qu'il s'agit d'une compression **sans perte**, donc l'algorithme restitue bit pour bit la séquence originale. On parle de **réversibilité**. Inversement, la compression avec perte se permet de supprimer des informations inutiles pour gagner encore plus d'espace, telles que certaines fréquences *inaudibles* pour l'Homme avec le MP3 ou le OGG (ce sont des formats *lossy*). Comparez une musique au format lossless et la même au format lossy, et je ne vous apprendrai rien en disant que vous constaterez une nette différence de poids, alors que la différence de qualité à l'oreille est discutable.

Parenthèse faite, suivant l'algorithme dont il est question, on parlera de plusieurs attributs  : 

- **le taux de compression** <=> volume final / volume initial %
- **la vitesse de compression** en MB/s
- **la vitesse de décompression** en MB/s
- **le coût CPU** engendré par les deux opérations ci-dessus

Pour illustrer la chose : certains algorithmes sont très rapides lors des opérations de compression et de décompression, alors que d'autres beaucoup moins performants dans ces opérations sont beaucoup plus efficaces pour obtenir un fichier compressé plus léger. Il faut donc choisir un algorithme adapté à la situation. Bien sûr, l'objectif en développant un nouvel algorithme comme Brotli est de faire en sorte qu'il soit à la fois plus efficace et à la fois plus rapide. Je voulais justement en venir au fait que, comme le suggère le titre de l'article, Brotli devrait avoir le vent en poupe concernant **son utilisation sur le Web** pour la compression HTTP. Aujourd'hui, gzip est répandu pour cette utilisation. 

*D'abord, en quoi consiste la compression des éléments d'une page Web ?*  Ou autrement dit, la [compression HTTP](https://en.wikipedia.org/wiki/HTTP_compression). Afin de sauver de la bande passante et en même temps accélérer les vitesses de transfert, le serveur peut compresser des éléments de la page avant de les envoyer. Si vous utilisez nginx, gzip devrait certainement être déjà configuré à cet effet. Si ce n'est pas le cas, il n'est pas trop tard pour voir [comment ça se configure](http://nginx.org/en/docs/http/ngx_http_gzip_module.html). Brotli ayant vocation à remplacer gzip, il n'est pas spécialement plus véloce, mais pour la même rapidité, son taux de compression est plus intéressant. Voici les chiffres publiés par Ilya Grigorik sur [Google+](https://plus.google.com/+IlyaGrigorik/posts/X9ogn4fLtHL), qui compare Brotli à gzip :

- **html** : gain de 25% pour Brotli
- **js** : gain de 17% pour Brotli
- **js minifié** : gain de 17% pour Brotli
- **css** : gain de 20% pour Brotli

Brotli est ici le grand vainqueur, sans aucun doute permis. Par curiosité, j'ai essayé de montrer par moi-même de façon empirique que Brotli a un meilleur taux de compression. J'espère que cette image sera suffisamment explicite pour vous :

![](https://pix.schrodinger.io/sUEQaCpX/CiwMYjhU.png)

**La grandeur utilisée pour la comparaison est ici le poids lors du transfert**. En effet, ça tient ses promesses : on peut estimer très grossièrement un gain général aux alentours des **15%**. J'ai veillé à utiliser des niveaux de compression **équivalents**, sans quoi ces valeurs n'auraient effectivement pas de sens. D'autres rapports, conclusions sur le sujet, et sans doute plus complets sont disponibles à ces adresses :

- [*Results of experimenting with Brotli for dynamic web content*](https://blog.cloudflare.com/results-experimenting-brotli/) de Vlad Krasnov.

- [*Comparison of Brotli, Deflate, Zopfli, LZMA, LZHAM and Bzip2 Compression Algorithms*](https://cran.r-project.org/web/packages/brotli/vignettes/brotli-2015-09-22.pdf), de Jyrki Alakuijala, Evgenii Kliuchnikov, Zoltan Szabadka, et Lode Vandevenne (Google Inc.).

Côté intégration sur nginx, il faut installer un module pour profiter de Brotli : **ngx_brotli**. Comme vous devez le savoir, il faut recompiler nginx avec pour en profiter. En plus de ça, il est nécessaire d'avoir une lib installée sur son système, qui est ~~étonnamment~~ **libbrotli**. Les pré-requis pour compiler libbrotli sont :

- libtool
- autoconf
- automake
- make
- git
- un compiler C

Dans `/tmp` par exemple, clonez le repo du projet pour build libbrotli:

```language
git clone https://github.com/bagder/libbrotli
```

Pour compiler et installer, il suffit de lancer :

```language
./autogen.sh && ./configure && make && make install
```

Enfin, il suffit de télécharger le module ngx_brotli et de compiler nginx avec :

```language-bash
mkdir /tmp/ngx_brotli && cd /tmp/ngx_brotli
wget -qO- https://github.com/google/ngx_brotli/archive/master.tar.gz | tar xz --strip 1
./configure --add-module=/tmp/ngx_brotli
```

Pour configurer Brotli, ça se passe dans `nginx.conf`, voici un exemple :

```language-nginx
    brotli on; #activer Brotli
    brotli_static on; #activer les fichiers pré-compressés
    brotli_buffers 16 8k; #nombre et taille des buffers
    brotli_comp_level 6; #niveau de compression (0-11)
    brotli_types #compression à la volée pour ces types MIME
        text/css
        text/javascript
        text/xml
        text/plain
        text/x-component
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/rss+xml
        application/vnd.ms-fontobject
        font/truetype
        font/opentype
        image/svg+xml;
```

Plus de détails évidemment sur la [page officielle](https://github.com/google/ngx_brotli/blob/master/README.md).
Alternativement, mon image Docker [*wonderfall/nginx*](https://hub.docker.com/r/wonderfall/nginx/) bénéficie déjà de Brotli. Pour rappel, tous mes Dockerfiles sont sur [Github](https://github.com/Wonderfall/dockerfiles).

**Note :** un juste milieu est à trouver pour la directive `brotli_comp_level` et il convient de vérifier les vitesses de transfert. L'outil d'inspection de Firefox/Chrome est intéressant pour cela, j'en détaille un peu l'utilisation par la suite.

Maintenant, vous allez sûrement avoir envie de tester, mais j'ai oublié de vous prévenir plus tôt : **Brotli ne fonctionne qu'avec les connexions HTTPS**. Je n'ai aucune idée de pourquoi, et si quelqu'un le sait, qu'il m'en fasse part via les commentaires. Je serais ravi d'apprendre la raison, même si finalement, ce n'est pas plus mal comme ça. Non ?

Bref, pour bien maîtriser vos essais, vous devez savoir une petite chose : lorsque vous demandez une page web, votre client adjoint dans sa requête des headers (en-têtes), comme par exemple le `User-Agent` que vous devez peut-être connaître... Un header dans la requête nous intéresse ici : `Accept-Encoding`. Ce header définit, pour résumer brièvement, les différentes méthodes de compressions HTTP supportées par le client.  Sur Chrome 48, c'est `accept-encoding:gzip, deflate, sdch`. Sur Firefox 44, c'est `Accept-Encoding :"gzip, deflate, br"`. Remarquez bien le `br` puisque c'est le diminutif de Brotli. Si vous avez suivi les news, Mozilla a bel et bien annoncé le [support de Brotli](https://www.mozilla.org/en-US/firefox/44.0/releasenotes/) dans la version 44 du navigateur au panda roux. Sur Chrome, il faut activer un flag accessible via `chrome://flags/#enable-brotli` présent sur le canal Canary, c'est d'ailleurs étonnant que Brotli ne soit pas encore présent sur le propre navigateur de Google. Passons ce détail...

C'est donc une histoire de headers avec laquelle on va pouvoir tester si tout ça fonctionne. On peut d'abord utiliser `curl` en envoyant le même header qu'utilise Firefox dans ses requêtes, par exemple :

```language
curl --header "Accept-Encoding: gzip,deflate,br" -I https://searx.schrodinger.io
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 02 Feb 2016 16:00:02 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Vary: Accept-Encoding
Strict-Transport-Security: max-age=15552000; includeSubdomains; preload
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Content-Encoding: br
```

On remarque bien le `Content-Encoding: br` retourné dans les headers de réponse, ce qui montre que le serveur Web supporte *a priori* Brotli. Si vous essayez avec un site qui ne supporte pas (encore) Brotli, vous obtiendrez sûrement `Content-Encoding: gzip`. 

C'est seulement un petit essai, et on n'a même pas essayé avec HTTP/2. Je propose qu'on aille directement dans Firefox : dans la fenêtre d'inspection, l'onglet *Réseau* nous intéresse. Puis on choisit un élément qui retourne le code 200 (*304 not modified* signifierait que l'élément est dans le cache, d'ailleurs là aussi il s'agit d'un jeu de headers), comme par exemple un css :

![](https://pix.schrodinger.io/rVgLb8Pu/rTnNgP0I.png)

Qu'est-ce qu'on voit ? Notre `Content-Encoding: br`. Youpi.

**Remarque** : si comme moi vous utilisez un nginx en reverse proxy et des applications derrière, il y a une seule règle à savoir. En fait, le module de Brotli ne compresse tout simplement pas un élément déjà compressé avec gzip. Il cohabite dans tous les sens avec gzip sans souci majeur. Une fois ce principe assimilé, c'est à vous d'adapter vos paramètres pour tirer profit ou non de Brotli. Par exemple :
 
- Back => reverse => **brotli** => envoi.
- Back => **gzip** => reverse => envoi.

Ce 1359e mot signe la fin de cet article. Si vous avez des remarques à me faire, n'hésitez pas tant que c'est constructif. Ou peut-être des demandes particulières dans la mesure du possible. J'essaie de trouver un équilibre dans mes explications pour que n'importe qui puisse comprendre aisément, mais si quelque chose ne va pas, je suis prêt à prendre note. Dans tous les cas, j'espère que ça sera utile à certains !
