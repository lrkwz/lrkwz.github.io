---
layout: post
title: Volumi docker, come proteggerli dalla cancellazione accidentale?
date: '2020-07-08'
author: Luca Orlandi
tags:
- docker
---

La gestione di progetti con docker è decisamente affascinante e potente, al momento lo sto utilizzando soprattutto con diversi team di sviluppo pertanto è spesso necessario _fare pulizia_ dei vari residui di volumi, reti, container e immagini; per questo la sequenza di comandi:

```
docker container prune -f
docker image prune -f
docker volume prune -f
docker network prune -f
```

fa il suo bel mestiere.

Sorge però un dubbio: come faccio a preservare quei volumi che **devono** essere conservati?

La soluzione che ho individuato consiste nell'applicare una `label` al volume e istruire docker volume prune acciocchè non consideri i volumi con la label prescelta.

Per esempio utilizzando un docker-compose.yml:

```
version: '3'
(...)
volumes:
  sharedvolume:
    labels:
      do-not-delete: true
```

a questo punto il comando `docker volume prune --filter label!=do-not-delete` elimina tutti i volumi ad eccezione del nostro.

Per farla più semplice si può aggiungere in `$HOME/.docker/config.json` il parametro `"pruneFilters": ["label!=do-not-delete"]`.

Q. E se volessi effettivamente eliminare anche quel volume?

A. Esegui `docker volume prune --filter label=do-not-delete`