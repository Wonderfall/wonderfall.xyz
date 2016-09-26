Quand bien même vous ne seriez pas un sysadmin, vous devez certainement savoir à quel point la configuration de services importants tels que SSH, plus précisément [OpenSSH](http://www.openssh.com/) (le plus souvent) qui utilise ledit protocole, est important.

Je ne veux pas réinventer la roue, ici il s'agira simplement de quelques astuces, et certaines peuvent s'apparenter à de la *sécurité par l'obscurité*. Il existe de nombreux tutoriels sur Internet pour bien configurer OpenSSH, voici une *to-do list* très sommaire :

- Désactiver la possibilité de se connecter en root, c'est **extrêmement conseillé**. Peut-être que ce conseil n'est pas si extrême que ça quand on considère une authentification très sécurisée, mais de manière générale, je n'exposerais jamais root à un service accessible depuis l'extérieur.

- Autoriser le minimum d'utilisateurs à pouvoir se connecter, au mieux un seul utilisateur dédié.

- Changer le port (22 par défaut), utile pour éviter les nombreux bots dans les logs.

- Utiliser l'authentification par clés. C'est bien plus efficace qu'un mot de passe.

Ces quatre conseils sont à appliquer en priorité selon moi. Elles permettront d'éviter les nombreux bots, tout en renforçant la sécurité de façon globale. Pour faire court : **la configuration par défaut doit être changée**. Autre piqûre de rappel : maintenez à jour OpenSSH. C'est important. Et je ne peux pas m'empêcher de faire une référence à [cet excellent article](https://stribika.github.io/2015/01/04/secure-secure-shell.html) qui vous donnera de multiples crypto-astuces. 

J'en profite pour faire un aparté qui concerne **SFTP**. SFTP permet la communication et l'échange de fichiers à la manière de FTP (même si ça n'a rien à voir en tant que protocole) à travers le protocole SSH. La configuration du SFTP se fait donc via OpenSSH. Il est recommandé de faire un SFTP *chrooté*, c'est-à-dire que l'utilisateur connecté en SFTP ne pourra pas accéder à `/` : il aura seulement accès à un certain emplacement qui sera pour lui la racine (impossible d'aller plus loin), généralement son home. C'est d'autant plus intéressant quand vous avez plusieurs utilisateurs sur votre machine et que vous ne souhaitez pas qu'ils aillent fouiller un peu partout... Naturellement, il n'est plus question d'utiliser  une connexion SSH classique avec cet utilisateur.

Revenons-en à notre configuration d'OpenSSH. Le fichier de configuration se situe généralement dans `/etc/ssh/sshd_config`, c'est le cas pour Debian par exemple. Quelque chose qui n'est pas forcément évident pour un débutant, c'est la mise en place des clés SSH. *Mais d'ailleurs, pourquoi utiliser des clés plutôt que des mots de passe ?* Un premier argument est que justement, vous n'avez **plus de mot de passe à saisir** (ou à copier/coller). Pas de mot de passe à intercepter non plus. Le second argument est la diminution des risques d'une attaque de type **brute-force**. Allez-y pour brute-force une clé... C'est très souvent, si ce n'est systématiquement, peine perdue pour l'attaquant.

Il s'agira ici d'une authentification asymétrique. Il faut générer une paire de clés : une **clé publique** et une **clé privée**. Sur votre PC, vous devez avoir `ssh-keygen` à votre disposition. Ça ne doit pas être un souci sur GNU/Linux, et OS X. Quant à Windows et toutes les autres subtilités, Github a [une page](https://help.github.com/articles/generating-ssh-keys/) pour vous expliquer comment utiliser `ssh-keygen`. Il est actuellement recommandé d'utiliser des clés **ed25519**, qui est un schéma de signature numérique se reposant sur la [cryptographie sur les courbes elliptiques](https://fr.wikipedia.org/wiki/Cryptographie_sur_les_courbes_elliptiques). Par rapport à d'autres variantes de courbes elliptiques et RSA, l'utilisation de ed25519 promet de **meilleures performances** et une **meilleure sécurité**. Voici comment générer tout simplement une paire de clés ed25519 :

```language
ssh-keygen -t ed25519
```

Il est vivement recommandé de protéger la clé par une phrase de passe. Si la clé tombe entre de mauvaises mains, c'est une **protection supplémentaire**. Vous devriez normalement obtenir deux fichiers : `id_ed25519.pub` et `id_ed25519`. Sur OS X, leur emplacement est par défaut dans `~/.ssh`. Maintenant il faut envoyer la clé publique au serveur, pour l'utilisateur souhaité (sur le serveur) :

```language
ssh user@server -p port "echo $(cat ~/.ssh/id_ed25519.pub) >> .ssh/authorized_keys"
```

Si le répertoire `~/.ssh` n'est pas présent du côté serveur (toujours pour l'utilisateur pour lequel on veut se connecter avec nos clés), il faut le créer. `~/.ssh/authorized_keys` est automatiquement lu et interprété par OpenSSH comme un emplacement classique où se situent les clés publiques. Bien évidemment, il faut essayer. Vous pouvez même passer `PasswordAuthentication` sur `no` quand tout est bon, pour complètement désactiver l'authentification par mot de passe. 

**N'oubliez pas, quand vous configurez SSH :** ne vous déconnectez JAMAIS de votre session en cours tout de suite après avoir redémarré le service. Ouvrez une autre session pour essayer avant vos dernières modifications. Comme ça, vous aurez toujours la main sur le serveur si SSH est devenu par accident inutilisable.
___
Passons maintenant à un autre élément : le 2FA, pour 2-Factor-Authentication. Vous connaissez sans doute déjà le principe, c'est-à-dire qu'on utilise plusieurs... méthodes d'authentification. Vous pouvez déjà le faire, ou l'avez déjà fait, en cumulant mot de passe et clés. Le 2FA en vogue est la génération d'un **code éphémère de 6 chiffres**, comme le fait **Google Authenticator**, une implémentation très connue de TOTP (*Time-based One-time*). La génération se fait indépendamment des deux côtés, aucune connexion n'est requise entre le générateur de code et le serveur. La synchronisation est possible car les deux ont initialement échangé une clé secrète et un compteur. À chaque incrémentation, un nouveau code existe à la fois pour le générateur et à la fois pour le serveur. Ce principe repose sur l'agorithme [TOTP](https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm).

Tout se déroule essentiellement sur votre serveur pour l'installation et la configuration. Commençons par installer les dépendances :

```language
apt-get install libpam-google-authenticator
```

Ensuite, il faut éditer le fichier `/etc/pam.d/sshd` qui est responsable des modules d'authentifications *pluggables* pour OpenSSH. Tout en haut du fichier, ajoutez `auth required pam_google_authenticator.so`.
Commentez cette ligne (ajoutez un `#` devant) : `@include common-auth`

Si on ne décommente pas cette ligne, on cumulera les clés, le générateur de code, et en plus de ça le mot de passe. On a donc du MFA avec authentification par mot de passe, or on souhaite s'affranchir de l'utilisation d'un mot de passe pour les raisons énoncées au début de l'article.

Pour la suite, tout se passe dans `/etc/ssh/sshd_config`. Changez la directive `ChallengeResponseAuthentication` pour qu'elle corrresponde à `yes`. En dessous, sur une nouvelle ligne, ajoutez `AuthenticationMethods publickey,keyboard-interactive:pam`. C'est plutôt explicite, car on veut utiliser nos clés, et interagir avec le F2A ensuite. Enfin, changez également `UsePAM` pour que ça soit mis sur `yes`.

Si vous souhaitez utiliser le 2FA dont on a parlé, il faut lancer l'utilitaire `google-authenticator` pour chaque utilisateur qui veut en bénéficier. Vous devriez obtenir ceci en le lançant :

![](https://pix.schrodinger.io/hweHqFGx/PTkEAcjc.png)

*(Il va de soi que les codes d'urgence sont à conserver précieusement.)*

Sur votre smartphone, vous devez avoir Google Authenticator d'installé. Pas de panique, l'application est open-source, même si c'est estampillé Google. Elle est disponible sur le Play Store et sur [F-Droid](https://f-droid.org/repository/browse/?fdfilter=two+factor&fdid=com.google.android.apps.authenticator2). Pour que la lecture du QR se fasse, il faut un lecteur à cet effet. [QR Droid](https://play.google.com/store/apps/details?id=la.droid.qr&hl=fr) remplit parfaitement ce rôle. Dans le menu de Google Authenticator, choisissez *Set up account*, puis *Scan a barcode*, et visez le code affiché par l'utilitaire. Le compte s'ajoute automatiquement dans votre liste sur l'application. En réalité, vous pouvez totalement utiliser une autre application qui a une implémentation de TOTP pour du 2FA, et il en existe sur PC à toutes les sauces.

**EDIT (16/05/16)** : En fait, la version de Google Authenticator présente sur le Play Store n'est pas open-source. La dernière libération du code remonte à un moment, vous trouverez une version maintenue [ici](https://github.com/google/google-authenticator-android). La version sur F-Droid est vieille car c'est une version open-source. À la place, j'utilise désormais [FreeOTP](https://f-droid.org/repository/browse/?fdid=org.fedorahosted.freeotp).

Continuons à configurer l'utilitaire. Je vous conseille de répondre ainsi, mais adaptez à vos besoins :

- **y**es : mettre à jour le `.google-authenticator`
- **y**es : désactiver l'utilisation de plusieurs codes
- **n**o : ne pas compenser les temps de synchronisation
- **y**es : activer le rate-limiting.

Une fois que c'est fait... Il n'y a plus qu'à tester. Lancez une autre session, connectez-vous sur l'utilisateur avec lequel vous avez lancé l'utilitaire. L'authentification par clés se fait bien, mais on vous demande quelque chose de nouveau en plus :

```language
Authenticated with partial success.
Verification code: 
```

Le code est trouvable dans l'application Google Authenticator. Il est composé de 6 chiffres, et il est éphémère : un nouveau code est généré par défaut toutes les 30 secondes.

![](https://pix.schrodinger.io/2HXHBbEx/6CJrRbgC.png)

Il suffit alors de rentrer le bon code pour que tout fonctionne, l'authentification sera alors complète.

*N.B.* : cela ne semble pas fonctionner avec SFTP *chrooté*. Il faut que `/home/user` appartienne à `user` et non à `root`. Dans la section `Match user` dans `sshd_config`, songez à adapter vos paramètres d'authentification (ils primeront sur les paramètres généraux si vous les redéfinissez dans ce bloc) pour désactiver le 2FA. Concrètement, il faut rajouter `AuthenticationMethods publickey` de façon correctement indentée par rapport au bloc `Match user`.
___
Voici une autre astuce, qui est celle du **port knocking**. Le port knocking est tout simplement un mécanisme qui s'enclenche quand une certaine séquence de ports est *toquée*. Cette détection se fait grâce aux logs du système (syslog), qui enregistrent quelle IP s'est connectée sur tel port.

Un pré-requis pour que le système puisse bien fonctionner est la fermeture des ports inutilisés. Typiquement, on utilisera la politique **DROP** pour la chaîne INPUT, si on utilise iptables. Les paquets sont ainsi jetés par le serveur, et ceci est notifié dans les logs, avec les ports concernés et l'IP à l'origine des connexions. Le daemon que nous allons utiliser, `knockd`, lit ces logs et reconnaît ainsi la séquence de ports si elle est *toquée*.

On souhaite par exemple qu'après cette détection, une règle permettant la connexion par SSH pour une certaine IP (la nôtre de préférence) soit ajoutée dans iptables, puis qu'après 10 secondes, cette règle soit automatiquement supprimée. Si vous réussissez à vous connecter, vous serez toujours connecté puisque la connexion sera établie, une règle essentielle est à ajouter pour ça : 

```language
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Avec `knockd`, tout ceci est possible. Installez le daemon :

```language
apt-get install knockd
```

Configurez `/etc/default/knockd` de cette façon :

```language
START_KNOCKD=1 # activer knockd
KNOCKD_OPTS="-i eth0" # interface d'écoute
```

Remplacez `eth0` par l'interface réseau que vous utilisez pour SSH. Normalement, c'est `eth0` dans la plupart des cas. Il faut d'ailleurs bien vérifier que `knockd` se lance sur cette interface, j'ai déjà eu des soucis avec ça.

La configuration du démon se déroule dans `/etc/knockd.conf` :

```language
[options]
    UseSyslog
    interface = eth0

[SSH]
    sequence    = ...
    seq_timeout = 15
    tcpflags   = syn
    start_command = /sbin/iptables -A INPUT -s %IP% -p tcp --dport PORT -j ACCEPT
    cmd_timeout = 10
    stop_command  = /sbin/iptables -D INPUT -s %IP% -p tcp --dport PORT -j ACCEPT
```

**PORT** est à remplacer 2 fois ici : c'est logiquement le port que vous utilisez pour OpenSSH, par défaut 22. Vous pouvez bien sûr adapter les commandes. Il faut compléter la séquence avec vos ports, choisissez-les **aléatoirement**. Vous pouvez également **alterner udp et tcp**. Les ports sont séparés par des virgules, les protocoles sont quant à eux précisés avec un double point après le port. **Par exemple :** `2083,1038:udp,23824`. N'hésitez pas à changer `cmd_timeout`, c'est le délai à partir duquel la règle permettant la connexion à SSH est supprimée. Une valeur plus importante sera donc plus adaptée pour les connexions lentes. 

N'oubliez pas de redémarrer le service et de faire vos essais. *Au fait, comment faire cette séquence de ports plus simplement depuis le client ?* On peut utiliser `nmap`, mais `knock` (et pas `knockd` qui est le daemon installé sur le serveur) permet cela très aisément. Un example vaut mieux qu'une longue explication :

```language
knock serveur 2083 1038:udp 23824 && ssh user@serveur -p port
```

Ainsi, votre port SSH est complètement **invisible** aux yeux des scans. C'est un classique de la *sécurité par l'obscurité* dont l'intérêt est perpétuellement discuté. Mais combinez ça à l'utilisation de honeypots, tels que Kippo ou encore Cowrie, et vous verrez les attaquants tomber dans le panneau encore plus facilement. Cela fera peut-être l'objet d'un futur article, tout comme la configuration d'iptables.
