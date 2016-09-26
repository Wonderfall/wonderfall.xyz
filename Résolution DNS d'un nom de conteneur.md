Dans certains cas il peut s'avérer intéressant de pouvoir communiquer avec un conteneur Docker sans devoir récupérer son IP manuellement en faisant une inspection du conteneur :

```language-bash
docker inspect conteneur | grep IPAddress\" | head -n1 | grep -Eo "[0-9.]+"
```

D'autant plus que quand le conteneur est recréé ou même redémarré, son IP est susceptible de changer, et c'est loin d'être l'idéal dans le cas où l'on doit préciser l'adresse du conteneur dans une configuration fixe, par exemple pour nginx (si on l'utilise sur l'hôte) avec la directive `proxy_pass`. De façon similaire à ce que fait Docker en liant des conteneurs entre eux, je voulais une solution automatisée qui puisse rajouter des entrées dans `/etc/hosts`. 

C'est un peu casse-tête à faire, mais je me suis rendu compte qu'il existait plus simple à base de DNS. En effet, un petit serveur DNS nommé **resolvable** développé par Gliderlabs, et qui tourne dans un conteneur va pouvoir accomplir cette tâche. Il suffit de lui lier le socket Docker (à éviter en général) et éventuellement `/etc/resolv.conf` pour qu'il puisse s'y rajouter tout seul avec une entrée `nameserver`. C'est aussi simple que cela finalement :

```language-bash
docker run -d \
      --hostname resolvable \
      -v /var/run/docker.sock:/tmp/docker.sock \
      -v /etc/resolv.conf:/tmp/resolv.conf \
      gliderlabs/resolvable:master
```

Et en version compose :

```language-yaml
resolver:
  image: gliderlabs/resolvable:master
  container_name: resolver
  hostname: resolvable
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock
    - /etc/resolv.conf:/tmp/resolv.conf
```

Vos conteneurs seront accessibles depuis l'hôte en résolvant `conteneur.docker`. Par exemple avec mon conteneur Emby, je vais pouvoir tester si ça fonctionne bien comme prévu :

```language-bash
$ ping -c 1 emby.docker
PING emby.docker (172.17.0.2) 56(84) bytes of data.
64 bytes from e24d4dc35996 (172.17.0.2): icmp_seq=1 ttl=64 time=0.048 ms

--- emby.docker ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.048/0.048/0.048/0.000 ms

$ dig +short emby.docker
172.17.0.2
```

Et du coup, utiliser `emby.docker` pour faire un reverse avec nginx (s'il est installé sur l'hôte) :

```language-nginx
location / {
  proxy_pass http://emby.docker:8096;
  include /etc/nginx/conf.d/proxy_params.conf;
}
```

Si vous voulez en savoir plus, allez jeter un oeil sur [Github](https://github.com/gliderlabs/resolvable) ; resolvable a d'autres features intéressantes qui pourraient vous intéresser.

Mais cette solution a plusieurs inconvénients... D'abord il convient de l'ajouter en tête des nameservers, et comme resolvable ne résoudra que les conteneurs et pas autre chose, il va falloir attendre le fallback vers un autre nameserver pour résoudre un nom de domaine classique. On peut toujours diminuer ce temps... mais c'est loin d'être optimal. Donc je me suis finalement rabattu sur la solution à base de `/etc/hosts`. J'ai créé un script à cet effet :

```language-bash,line-numbers
#!/bin/bash
echo "Adding containers to /etc/hosts..."
sed -i -n '/DOCKER CONTAINERS/q;p' /etc/hosts
echo "# DOCKER CONTAINERS" >> /etc/hosts
NUMBER=$(docker info | grep "Running:" | tr -dc '0-9')+1

for ((n=1;n<$NUMBER;n++))
do
   CONTAINER=$(docker inspect --format='{{.Name}}' $(docker ps -aq) | sed -n ${n}p | tr -d /)
   IP=$(docker inspect ${CONTAINER} | grep IPAddress\" | head -n1 | grep -Eo "[0-9.]+")

   echo "${IP} ${CONTAINER}.docker" >> /etc/hosts
done

echo "Done."
```

Il y a encore un inconvénient, il faudrait lancer le script à chaque fois qu'un conteneur redémarre ou est recréé. On ne le fait pas forcément tous les jours en production, mais j'aimerais bien trouver un moyen de résoudre ce problème. Je mettrai à jour l'article quand ce sera le cas. Un début de solution serait d'utiliser des IPv4 statiques : on créer un nouveau réseau de conteneurs avec le driver `bridge`, on choisit un subnet et il n'y a plus qu'à assigner à un conteneur ce réseau et son IP statique. Mais pour `docker-compose` cela nécessiterait l'utilisation d'un [compose file](https://docs.docker.com/compose/compose-file/) v2 ; et je ne me suis pas encore penché sur le sujet.
