---
layout: post
title: How to properly install docker-compose
date: '2016-02-28'
author: Luca Orlandi
tags:
- docker
- docker-compose
- ubuntu
---
Non c'è partita, a questo giro [Docker](http://www.docker.com) batte tutti a mani basse: si tratta senza dubbio della tecnica più interessante per definire ambienti di sviluppo, testing e financo produzione (con questa ancora non ho giocato ma poco ci manca).

Dopo un po che studi la documentazione diventa evidente che lo strumento fondamentale per automatizzare la definizione degli ambienti è [docker-compose](https://docs.docker.com/compose/).

Il sito di Docker purtropo dà delle [istruzioni](https://docs.docker.com/compose/install) sbagliate per la sua installazione; nulla di troppo grave per carità: semplicemente le istruzioni fanno scaricare il binario del comando con curl per salvarlo direttamente in ```/usr/local/bin``` dove evidentemente solo root può scrivere. Se precedi il tutto con ```sudo -s``` sei a cavallo.
Se invece vuoi più elegantemente installare docker-compose con una sola riga di comando puoi fare così:

{% highlight shell %}
curl -L https://github.com/docker/compose/releases/download/1.6.2/docker-compose-`uname -s`-`uname -m` | sudo tee /usr/local/bin/docker-compose > /dev/null
{% endhighlight %}

evidentemente devi sostituire a 1.6.2 il numero dell'ultima versione se vuoi restare aggiornato.
