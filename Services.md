#### Préambule
Je mets à libre disposition du public, et sous sa propre responsabilité, plusieurs services que j'héberge moi-même sur mon serveur. Il n'y aucune garantie à leur utilisation, c'est-à-dire qu'un service peut être coupé du jour au lendemain, et je m'en réserve particulièrement le droit dans le cas où un service est détourné de sa fonction première et utile.

Je peux dans tous les cas promettre que je me garde bien d'être gourmand vis-à-vis des données personnelles. Aucun tracking, aucun mouchard. De plus, certains services ont recours au chiffrement, ce qui fait que je ne peux pas lire les fichiers depuis mon serveur directement.

J'utilise systématiquement **DNSSEC** et **DANE**. Si au moins l'un des deux protocoles n'est pas respecté, alors posez-vous des questions sur l'authenticité de l'instance visitée. Il existe une extension sur Firefox, [DNSSEC/TLSA Validator](https://www.dnssec-validator.cz/), pour vérifier cela.

#### PrivateBin
PrivateBin est un pastebin-like, originellement un fork de Zerobin (qui n'est plus maintenu), open-source et self-hostable, qui a la particularité de chiffrer les pastes de sorte à ce que le serveur ne puisse pas les voir en clair.

- Instance : https://paste.schrodinger.io
- Sources : https://github.com/PrivateBin/PrivateBin

#### Lutim
Lutim, pour Let's Upload That Image, est comme son nom l'indique de façon explicite un service permettant l'hébergement d'images. La durée de vie d'une image peut être définie. Les images sont chiffrées côté serveur.

- Instance : https://pix.schrodinger.io
- Sources : https://github.com/ldidry/lutim

#### Searx
Searx est un méta-moteur de recherche. Il permet de corréler plusieurs résultats issus de plusieurs moteurs de recherche différents, mais tout cela se fait anonymement pour vous.

- Instance : https://searx.schrodinger.io
- Sources : https://github.com/asciimoo/searx

Pour garantir l'efficacité de l'instance et la protéger des bots, j'ai choisi d'installer une authentification HTTP (user : **wonder**, mot de passe : **schrodingerscat**).
