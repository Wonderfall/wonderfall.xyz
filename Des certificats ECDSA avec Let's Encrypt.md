Vous devez sûrement connaître [Let's Encrypt](https://letsencrypt.org/) : une nouvelle [autorité de certification](https://fr.wikipedia.org/wiki/Autorit%C3%A9_de_certification) (CA) qui se dit ouverte, gratuite et automatisée. Aujourd'hui en bêta publique, Let's Encrypt vous propose de générer des certificats d'une longévité de [90 jours](https://letsencrypt.org/2015/11/09/why-90-days.html). Si vous souhaitez simplement générer un certificat sans prise de tête, je vous conseille de suivre [ce tutoriel](https://mondedie.fr/viewtopic.php?id=7414).

Le 10 Février dernier, Let's Encrypt a annoncé dans un [tweet](https://twitter.com/letsencrypt/status/697504441075798016) que l'autorité peut désormais signer des clés ECDSA. Des certificats dits ECDSA, donc. Pour information, Let's Encrypt génère automatiquement des certificats RSA actuellement, sans laisser le choix. Pour obtenir des certificats ECDSA, il faut mettre les mains dans la pâte... Mais ce n'est pas très compliqué non plus.

Alors, comme d'habitude (j'espère que vous vous posiez la question) : pourquoi faisons-nous cela ? En fait, ECDSA repose sur un autre problème mathématique que RSA ; ECDSA est une sous-famille de la grande famille des algorithmes de signature numérique qui font appel à la [cryptographie sur les courbes elliptiques](https://fr.wikipedia.org/wiki/Cryptographie_sur_les_courbes_elliptiques) (ECC). J'en ai déjà parlé dans un [précédent tutoriel](https://cats.schrodinger.io/securiser-ssh-cles-2fa-port-knocking/), ces algorithmes ont plusieurs avantages :

- Clés plus courtes, pour une sécurité comparable
- Moins de travail CPU pour chiffrer/déchiffrer
- Consommation de RAM amoindrie

C'est tout bénéfique, aussi bien du côté du serveur que du côté du client. Certains diront qu'on ne peut pas faire aussi bien confiance à ECDSA (on utilisera les courbes NIST) qu'à RSA, notamment parce que ce sont des suites cryptographiques issues de la NSA. Des [critiques](https://cr.yp.to/newelliptic/nistecc-20160106.pdf) vont notamment dans ce sens. On peut également noter que la NSA elle-même n'y croit plus, ECC n'étant pas *quantum proof* (tout comme RSA).

Nous allons illustrer l'article avec la courbe **secp384r1**, autrement appelée P-384, membre de la [suite cryptographique B de la NSA](https://www.nsa.gov/ia/programs/suiteb_cryptography/). Pourquoi P-384 ? Selon la [NSA elle-même](https://www.nsa.gov/ia/programs/suiteb_cryptography/), P-384 est idéale pour les documents *top secrets*. Et si vous êtes paranoïaque comme moi, ce serait parfait pour vous aussi. C'est apparemment le *niveau maximal* supporté par Let's Encrypt, croyez-moi, j'ai essayé avec P-521 mais ça n'a pas fonctionné (*P-521 not allowed*). Pour vous donner une idée, P-384 est équivalent en termes de sécurité à une clé RSA 7680 bits. Tout n'est pas rose non plus, car les courbes NIST [ne sont pas les plus sûres](http://safecurves.cr.yp.to/index.html) qui existent de nos jours. L'utilisation d'autres courbes pour ECDH et les certificats par exemple, est actuellement au simple stade du brouillon de leur spécification dans ce cadre.

Pour commencer, placez-vous dans un répertoire tout propre. On va maintenant créer la fameuse **clé** ECC P-384 :

```language
openssl ecparam -genkey -name secp384r1 > privkey.pem
``` 

Au passage, les différentes courbes elliptiques disponibles peuvent être listées via cette commande :
 
```language
openssl ecparam -list_curves
```

On peut maintenant créer un **fichier CSR**, qui contient toutes les informations relatives à une demande de certificat. Mais comme moi, vous aurez peut-être envie de générer un seul certificat pour plusieurs domaines, histoire de ne pas s'emmêler avec HPKP, TLSA, et compagnie. Vous étiez bien à l'aise avec le paramètre `-d` de Let's Encrypt... Dans cette éventualité, vous devez tout d'abord copier le fichier `/etc/ssl/openssl.cnf` dans votre répertoire de travail. Par exemple :

```language
cp /etc/ssl/openssl.cnf custom.cnf
```

On va éditer ce `custom.cnf` pour bénéficier d'un CSR multi-domaine. Voici comment.

Dans la section `[ req ]`, décommettez `req_extensions = v3_req`. Dans la section `[ v3_req ]`, ajoutez une nouvelle ligne : `subjectAltName = @alt_names`. Enfin, tout se passe dans `[ alt_names ]` pour ajouter vos domaines. Cette section n'existe pas, il faut la créer, par exemple tout à la fin du fichier. Pour chaque domaine souhaité, on l'associe à la directive `DNS.k` où `k`, allant de `1` à `n`, est incrémenté sur chaque nouvelle ligne. Le [SAN](https://en.wikipedia.org/wiki/SubjectAltName) que peut signer Let's Encrypt peut atteindre 100 (autrement dit, 100 domaines différents dans un seul certificat X.509), donc `n <= 100`.

Un exemple sera davantage explicite :

```language
[ alt_names ]
DNS.1 = sub1.domain.tld
DNS.2 = sub2.domain.tld
DNS.3 = sub3.domain.tld
DNS.4 = sub4.domain.tld
...
DNS.n = subn.domain.tld
```

Enregistrez toutes ces modifications. Vous pouvez désormais générer votre CSR avec la commande suivante :

```language
openssl req -new -sha256 -key privkey.pem -out csr.der -config custom.cnf
``` 

Rentrez des informations, mais laissez *Common Name* vide, et ne protégez pas avec un mot de passe. Vous devez, à ce stade, avoir en votre possession un `privkey.pem` et un `csr.der`.

Je pars du principe que vous savez déjà utiliser le client de Let's Encrypt. Si ce n'est pas le cas, je vous re-redirige vers [ce tutoriel](https://mondedie.fr/viewtopic.php?id=7414), et vers le [guide utilisateur](https://letsencrypt.readthedocs.org/en/latest/using.html#installation). J'ai personnellement choisi la méthode via Docker, mais j'ai eu quelques embrouilles avec des certificats générés dans `/opt/letsencrypt`... et évidemment, je n'ai pas monté un volume dessus. Faites donc attention ! 

Afin d'obtenir notre certificat à partir du CSR custom, on peut utiliser cette commande, en précisant si possible les chemins où LE disposera les fichiers générés et obtenus :

```language-bash
letsencrypt \
    certonly --standalone \
    --agree-tos \
    -m you@domain.tld \
    --csr /path/csr.der \
    --cert-path /path/cert.pem \
    --chain-path /path/chain.pem \
    --fullchain-path /path/fullchain.pem
```

Si tout va bien, Let's Encrypt vous indique que le certificat a été créé et son emplacement. Vous devriez avoir ces fichiers à votre disposition :

- **0000_cert.pem** : certificat du domaine
- **0000_chain.pem** : chaîne *of trust* Let's Encrypt
- **0001\_chain.pem** *ou* **0000\_fullchain** : cert + chain
- **privkey.pem** : clé privée

Je prendrai nginx en exemple pour la configuration et l'installation de nos nouveaux certificats. Ça peut quand même vous aider, peu importe si vous utilisez Apache, h2o, IIS, ou autre. Dans vos vhosts nginx, il va falloir bien évidemment spécifier le chemin vers le certificat complet et la clé :

```language-nginx
ssl_certificate /path/0001_chain.pem;
ssl_certificate_key /path/privkey.pem;
ssl_trusted_certificate /path/0000_chain.pem; # OCSP Stapling
```

Mais ce n'est pas tout : les **ciphers doivent être adaptés**, sinon la négociation TLS ne fonctionnera pas. Dans mon exemple, j'utilise cette suite *(ne la copiez pas bêtement !)* :

```language
ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256
```

Pour convertir votre *ancienne* suite (si elle est explicite comme la mienne), il suffirait de remplacer toutes les mentions à RSA par ECDSA. Je vous remets quand même à [cette page](https://tls.imirhil.fr/ciphers) qui est très utile. Naturellement, n'oubliez pas d'appliquer vos modifications avec, par exemple, `service nginx reload` ou `docker exec -ti nginx nginx -s reload` pour les convertis.

Plus qu'à tester ! Ça devrait déjà fonctionner dans votre navigateur... Sinon, utilisez [cet utilitaire](https://tls.imirhil.fr) pour être sûr que tout fonctionne ou diagnostiquer des erreurs. Dans mon exemple, cela donne :

![](https://pix.schrodinger.io/GJsCcaX4/fXpim5eD.png)

*Tout est au vert*, semble-t-il. J'espère qu'il en sera de même pour vous, et si ce n'est pas le cas, je suis à votre disposition quoiqu'il arrive.
