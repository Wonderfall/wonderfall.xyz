Coucou tout le monde. Vu que j'avais un peu de temps, j'ai trouvé intéressant de faire un petit article afin de vous faire découvrir [Flood](https://github.com/jfurrow/flood). Flood, c'est quoi ? Tout simplement une alternative à rutorrent, c'est à dire une interface web pour rTorrent. Flood repose sur des technos modernes avec node.js en backend et on retrouve du React du frontend. Donc sur le papier, c'est vraiment intéressant... aussi bien à travers les images :

![](https://camo.githubusercontent.com/d8f5cb502f06e0ea1cc171550c2bed035293c1a9/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6a6f686e667572726f772e636f6d2f73686172652f666c6f6f642d73637265656e73686f742d612d303630362e706e67)

L'interface est vraiment soignée, fluide, et je dirais qu'il y a l'essentiel. Flood n'est pas aussi complet que rutorrent. Pour le moment en tout cas, puisque le projet est encore jeune, et le développer semble ouvert aux requêtes. Par exemple je n'ai plus la possibilité de créer un .torrent pour l'uploader sur un tracker, mais à part ça je dois dire que rutorrent ne me manque aucunement. Flood fait peu de choses mais il le fait élégamment. On retrouvera également un système d'authentification intégré qui permettra de se passer de l'authentification HTTP nécessaire avec rutorrent.

Pour la partie installation, ça n'a rien de compliqué. Il faut s'assurer de disposer :

* D'une version de rTorrent avec XMLRPC.
* De node.js, dans sa version 4.x ou ultérieure.

Voilà, il suffira donc de cloner le projet et de lancer un `npm install --production`. Cela dit, j'ai dû installer la version de développement pour utiliser Flood avec node.js 6.x. L'étape de configuration est simple également, tout se déroule dans le fichier `config.js` où il suffira de configurer le port (ou le socket) SCGI correspondant à rTorrent.

Si vous utilisiez mon image Docker `wonderfall/rutorrent`, vous serez peut-être intéressés par `wonderfall/rtorrent-flood` disponible [ici](https://hub.docker.com/r/wonderfall/rtorrent-flood/). La transition se fait aisément, il faudra simplement penser à retirer la variable `WEBROOT` qui est devenue inutilisée, à utiliser la variable d'environnement `FLOOD_SECRET` utilisée pour [JWT](https://github.com/auth0/node-jsonwebtoken), et à faire persister le volume `/usr/flood/server/db`. Ah et il faudra également changer le port utilisé par le reverse proxy, Flood étant configuré par défaut pour utiliser le port 3000 et c'est ce que j'ai gardé pour l'image aussi.

C'est à peu près tout, je ne vois pas ce que je pourrais dire de plus, si ce n'est que j'espère que je vais faire découvrir Flood à certaines personnes qui vont tout autant apprécier que moi les finitions de cette interface.
