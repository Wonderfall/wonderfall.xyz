J'en faisais part dans mon précédent (et premier) billet, mais auparavant, j'utilisais [Disqus](https://disqus.com/). Disqus est une technologie très répandue pour intégrer facilement un système de commentaires sur un CMS qui n'en dispose pas, tel que [Ghost](https://ghost.org/). Le problème, c'est que Disqus est centralisé sur leurs serveurs, et évidemment, ce n'est pas open-source.

On n'en doute pas, il existe quelques alternatives à Disqus : [Discourse](https://www.discourse.org/), [Juvia](https://github.com/phusion/juvia), [**Isso**](https://posativ.org/isso/), et certainement [d'autres](http://alternativeto.net/software/disqus/). Isso, écrit en Python, m'a tout de suite tapé dans l'oeil, car contrairement à certaines alternatives, Isso s'installe sans prise de tête et est très léger (de ce fait beaucoup moins complet). Les commentaires sont rédigés en [Markdown](https://fr.wikipedia.org/wiki/Markdown), la base de données utilise SQLite, et l'intégration se résume à l'ajout d'un bout de code dans le thème de Ghost, plus précisément dans `post.hbs` (casper).

Comme je suis friand de Docker, j'ai tout de suite créé mon propre [dockerfile](https://github.com/Wonderfall/dockerfiles/tree/master/isso). Ce qui existait déjà sur le Hub n'était pas assez maintenu, propre, ou léger à mon goût. Les détails sont soit sur le [README](https://github.com/Wonderfall/dockerfiles/blob/master/isso/README.md) du Gihub ou sur la page du [Hub](https://hub.docker.com/r/wonderfall/isso/). La configuration se limite pour le moment à quelques lignes :

```language
[general]
dbpath = /db/comments.db
host = https://cats.schrodinger.io/
[server]
listen = http://0.0.0.0:8080/
```

Ensuite il ne reste plus qu'à ajouter un vhost pour utiliser le reverse proxy, et enfin, à intégrer mon instance Isso dans mon blog Ghost. Comme je l'ai dit, tout se passe dans `post.hbs`, à insérer où on le souhaite (en dessous de la balise `{{/author}}` par exemple) :

```language-css
<script data-isso="//comments.schrodinger.io/"
        data-isso-avatar="false"
        data-isso-vote="false"
        src="//comments.schrodinger.io/js/embed.min.js"></script>

<section id="isso-thread"></section>
```

C'est une configuration très sommaire mais qui fonctionne actuellement. ~~Je n'ai pas encore pensé au système de modération~~. De toute façon, je voulais un truc simple et qui fonctionne. Isso correspond à mes attentes pour cela. Je viens de le découvrir très récemment, donc l'instance actuellement intégrée sur ce blog risque de subir des modifications d'ici peu. Si vous souhaitez aussi vous y mettre, [la documentation](https://posativ.org/isso/docs/) vous sera utile.

**EDIT (05/02/16) :** C'est enfin configuré comme je le souhaitais. Désormais les notifications SMTP sont activées, et par ce même moyen la possibilité de modérer (la modération est donc assez sommaire mais c'est fonctionnel). J'en ai profité pour activer quelques protections contre le spam qui dissuaderont les malins de scripter un *while true; do curl...; done* pour citer la doc. 

```language
[general]
dbpath = /db/comments.db
host = https://cats.schrodinger.io/
max-age = 1h30m
notify = smtp

[server]
listen = http://0.0.0.0:8080/

[moderation]
enabled = true
purge-after = 30d

[smtp]
username = cat@schrodinger.io
password = xxxxxxxxxxxx
host = cat.schrodinger.io
port = 587
security = starttls
to = wonderfall@schrodinger.io
from = IssoUntitled <cat@schrodinger.io>
timeout = 10

[guard]
enabled = true
ratelimit = 2
direct-reply = 5
reply-to-self = false
```
