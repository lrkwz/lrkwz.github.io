---
layout: post
title: How to configure HTTPS for Drupal on Jelastic
date: '2016-04-26'
author: Luca Orlandi
tags:
- drupal
- jelastic
- ssl
---
Ancora una volta ho avuto la chance di apprezzare le feature di Jelastic: avevo bisogno di mettere rapidamente la versione di collaudo di un sito sotto HTTPS ed è stato possibile farlo con un semplice click.

Jelastic ha un certificato whildcard che supporta i domini ```*.<nomeprovider>.jelastic.com``` che rende la configurazione assolutamente brainless; altrimenti non devi fare altro che caricare il tuo certificato.

L'HTTPS è gestito sul frontend pertanto il webserver non necessita di alcuna configurazione.

Se da un lato questo è estremamente comodo (e corretto da un punto di vista architetturale a mio modo di vedere) dall'altro impedisce all'applicazione di fare le sue considerazioni automaticamente: drupal in particolare nella sua funzione ```drupal_settings_initialize()``` considera la valorizzazione di ```$_SERVER['HTTPS']``` cosa che avviene solo quando l'HTTPS arriva fino all'engine PHP. Di conseguenza il base url generato automaticamente da Drupal in questo contesto è sempre HTTP://.

Per ovviare al problema è sufficiente aggiungere al file di configurazione ```sites/default/settings.php``` la dichiarazione della base url:

    $base_url = 'https://yousite.example.com'

a questo punto la inizializzazione non tiene più solo conto della variabile di ambiente $_SERVER bensì *spacchetta* la base url dichiarata per prelevarne lo schema.