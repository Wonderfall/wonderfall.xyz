**BOINC**, c'est l'acronyme de *Berkeley Open Infrastructure for Network Computing*. Mais c'est plus concrètement une plateforme de calcul réparti (par projets) basé sur le principe du volontariat. Vous allouez des ressources de votre machine à [un des projets de calcul proposés](https://boinc.berkeley.edu/projects.php). On y trouve de la recherche en astrophysique, mathématiques, cryptographie, ou encore médecine.

![logo boinc](https://upload.wikimedia.org/wikipedia/commons/thumb/7/72/BOINC_logo_July_2007.svg/220px-BOINC_logo_July_2007.svg.png)

BOINC étant une plateforme, nous nous intéresserons ici à la mise en place rapide et simple d'un client. Et je ne vais pas y aller par quatre chemins : **Docker sera de la partie**, et sera donc le seul pré-requis peu importe la plateforme que vous utilisez. L'image que nous allons utiliser sera celle-ci : [wonderfall/boinc](https://hub.docker.com/r/wonderfall/boinc/). Basée sur Alpine Linux (*as always*), BOINC y est compilé depuis les sources. Récupérons l'image depuis le Docker Hub :

```language-bash
docker pull wonderfall/boinc
```

Puis créons un dossier pour la persistence des données, avec les permissions de l'utilisateur `boinc` de l'image :

```language-bash
mkdir /mnt/docker/boinc
chown 35854:35854 /mnt/docker/boinc
```

Ce qu'on peut dores et déjà faire, c'est configurer le client BOINC, même si ce n'est pas nécessaire. [Cette page](https://boinc.berkeley.edu/wiki/Client_configuration) de la documentation liste tous les paramètres de configuration. La configuration sera lue depuis un fichier XML : `/mnt/docker/boinc/cc_config.xml`. Le format suivant doit être impérativement suivi :

```language-xml
<cc_config>
   <log_flags>
       [ ... ]
   </log_flags>
   <options>
       [ ... ]
   </options>
</cc_config>
```

Le point intéressant, je pense, réside dans la limitation des ressources. Il est par exemple possible de limiter le nombre de coeurs utilisés avec `<ncpus>N</ncpus>`, mais comme nous utilisons Docker, je préfère le faire directement au niveau du conteneur. Pour ce faire, Docker utilise les **cgroups**, fonctionnalité du noyau Linux permettant, entre autre, un fin contrôle des ressources (CPU, RAM, bande passante, etc.) utilisables par un conteneur. BOINC consomme déjà "intelligemment" puisqu'il met en suspension ses tâches quand la charge CPU excède un certain pourcentage. Il ne devrait donc pas impacter les performances du serveur en cas de charge intensive (d'origine autre que BOINC). Mais on peut tout de même vouloir limiter le nombre de coeurs utilisables par le conteneur, et on peut le faire ansi par exemple :

```language-bash
docker run -d --cpuset-cpus="0,1" wonderfall/boinc
```

Le conteneur créé ne pourra donc utiliser que les coeurs 0 et 1 du CPU, donc deux au total. Bien sûr ces réglages dépendent de ce que vous avez dans votre machine et de ce que vous voulez allouer. Limiter le nombre de coeurs est la façon la plus directe de limiter la consommation de BOINC, mais vous pouvez aussi jouer avec les proportions de cycles CPU.

Bref, pour lancer le conteneur, voici une commande :

```language-bash
docker run -d --cpuset-cpus="0,1" --name boinc -v /mnt/docker/boinc:/home/boinc -h schrodingerscat wonderfall/boinc
```

... et son équivalent en **docker-compose** :

```language-yaml
boinc:
  image: wonderfall/boinc
  container_name: boinc
  hostname: schrodingerscat
  cpuset: 0,1
  volumes:
    - /mnt/docker/boinc:/home/boinc
```

Pensez à **adapter** les valeurs :

- **hostname** : ce que vous voulez, ça permet d'identifier votre serveur sur des plateformes au lieu de voir le hostname aléatoire généré par Docker.
- **cpuset** : cf. plus haut, c'est pour limiter le nombre de coeurs utilisables par le conteneur.
- **volumes** : `/mnt/docker/boinc` qui correspond à l'endroit sur l'hôte créé tout à l'heure pour être lié au `/home/boinc` du conteneur.

Une fois le conteneur lancé, on peut vérifier qu'il fonctionne correctement :

```language-bash
$ docker logs boinc
25-Jun-2016 13:31:26 [---] Starting BOINC client version 7.7.0 for x86_64-pc-linux-gnu
25-Jun-2016 13:31:26 [---] This a development version of BOINC and may not function properly
25-Jun-2016 13:31:26 [---] log flags: file_xfer, sched_ops, task
25-Jun-2016 13:31:26 [---] Libraries: libcurl/7.49.1 OpenSSL/1.0.2h zlib/1.2.8 libssh2/1.7.0
25-Jun-2016 13:31:26 [---] Data directory: /home/boinc
...
25-Jun-2016 13:31:26 Initialization completed
```

Bravo ! Maintenant il suffit de rejoindre un projet. Pour montrer un exemple, j'ai choisi le [World Community Grid](https://secure.worldcommunitygrid.org/) pour m'investir dans des projets de recherche sur les cancers, les virus (SIDA, Zika, Ebola), et les mystères du génome. 

![World Community Grid](https://pix.schrodinger.io/kt3OUmQr/gu09uO8I.png)

Il faut tout d'abord [s'inscrire](https://secure.worldcommunitygrid.org/join.action#signup) sur cette plateforme. Une fois son compte créé, on va dans Settings puis dans [My Profile](https://secure.worldcommunitygrid.org/ms/viewMyProfile.do) où il sera peut-être demandé de se connecter à nouveau. La section qui nous intéressante est celle-ci : 
*The following information is useful for attaching devices to World Community Grid*. Récupérez la chaîne de caractères qui correspond à **Account Key**. De retour sur le serveur, on va devoir rentrer *dans* le conteneur pour pouvoir utiliser `boinccmd` qui est un outil en ligne de commande permettant la configuration du client BOINC :

```language-bash
docker exec -ti boinc sh
```

`boinccmd` dispose de plusieurs paramètres intéressants que je ne vais pas développer dans cet article, vous les trouverez [ici](https://boinc.berkeley.edu/wiki/Boinccmd_tool). Celui qui nous intéresse permet de rejoindre un projet en fournissant l'URL de celui-ci et un moyen d'authentification, ici notre **Account Key** récupéré précédemment. Pour rejoindre le World Community Grid, rien de difficile :

```language-bash
boinccmd --project_attach www.worldcommunitygrid.org account_key
exit
```

Et voilà, normalement BOINC doit commencer à travailler, vous pouvez vérifier ça facilement avec :

```language-bash
docker exec -ti boinc boinccmd --get_state
```

Même si c'est loin de ce que peut faire un supercalculateur ou un ordinateur quantique, c'est toujours un petit coup de pouce pour la recherche !
