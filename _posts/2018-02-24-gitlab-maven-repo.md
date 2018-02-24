---
layout: post
title: Come implementare un repository maven in gitlab
date: '2018-02-24'
author: Luca Orlandi
tags:
- maven
- gitlab
- automazione
---
Forse non ho ancora avuto l'occasione di scriverlo ma lo ho detto sicuramente a tutti coloro i quali hanno la sfortuna di lavorare con me: [Gitlab](https://gitlab.com) in questo momento è sicuramente il servizio di hosting per lo sviluppo migliore che abbia mai individuato:

* repository pubblici e privati senza limite di dimensione e quantità
* issue tracking
* wiki
* docker image repository (!)
* continuous integration engine

(poi integrazione fra issue e repository git tramite i commenti delle commit, tra CI e repository con deploy e tag, ...)

Ad una analisi superficiale manca solo una implementazione di **repository maven** e poi sarebbe il sistema perfetto per ospitare tutti i miei sviluppi sia open-source che professionali.

Questo limite ha una soluzione tutto sommato piuttosto semplice.

Cercando sulla Rete puoi trovare sicuramente diversi articoli che illustrano come usare le estensioni del plugin `wagon maven` che si occupa del deploy degli artifact in modo che possano essere trasferiti tramite [git](https://git-scm.com) su [Github](https://github.com) o [Bitbucket](https://bitbucket.org). Ho trovato diverse estensioni adatte a questo scopo e ne ho scelta una: [Synergian wagon-git](http://synergian.github.io/wagon-git/) , non so se è la migliore ma funziona, le altre mi sono sembrate equivalenti. Quel che non si trova altrettanto facilmente è come utilizzare questi strumenti nello specifico di gitlab e soprattutto per progetti privati, sove l'accesso a qualunque risorsa è sottoposto ad autenticazione.

Per andare al sodo ho aggiunto al `pom.xml` della mia libreria comune il riferimento alla estensione di maven wagon

```xml
    <build>
    ...
        <extensions>
            <extension>
                <groupId>ar.com.synergian</groupId>
                <artifactId>wagon-git</artifactId>
                <version>0.3.0</version>
            </extension>
        </extensions>
    </build>
```
il repository dal quale scaricare l'estensione

```xml
    <pluginRepositories>
        <pluginRepository>
        <id>synergian-repo</id>
        <url>https://raw.github.com/synergian/wagon-git/releases</url>
        </pluginRepository>
    </pluginRepositories>
```

e infine le indicazioni di `distributionManagement` con le quali ho indicato quale repository voglio utilizzare per memorizzare **tutte** le mie librerie comuni:

```xml
    <distributionManagement>
        <repository>
            <id>garanteasy-repo</id>
            <name>Garanteasy repository</name>
            <url>git:releases://git@gitlab.com:garanteasy/maven-repo.git</url>
        </repository>
    </distributionManagement>
```
A questo punto il comando `maven deploy` compile impacchetta e trasferisce il jar all'interno del branch `releases` del repository mave-repo.

Chi mi conosce lo sa bene, sono fissato per la automazione: mai e poi mai permetterei l'uso diffuso di una libreria compilata e impacchettata sul pc di uno sviluppatore, quindi ovviamente ho adattato la descrizione del file `.gitlab.yml` che guida l'esecuzione del sistema di continuous integration in modo che al momento di una push + tag il jar venga compilato e trasferito automaticamente:

```yml
image: maven:3-jdk-8

variables:
  MAVEN_OPTS: -Dmaven.repo.local=${CI_PROJECT_DIR}/.m2

cache:
  paths:
    - .m2/repository

stages:
  - build
  - test
  - deploy

build:
  script:
    - mvn package

test:
  script:
    - mvn test

deploy:
  only:
    - tags
  script:
    - git config --global user.email "$GITLAB_USER_EMAIL"
    - git config --global user.name  "$GITLAB_USER_NAME"
    - mvn deploy -DperformRelease
```


dato che il progetto gitlab che ospita il repository maven è privato (come gli altri del resto) bisogna provvedere ad alimentare correttamente il meccanismo di autenticazione, per questo è sufficiente [generare una coppia di chiavi](https://docs.gitlab.com/ce/ssh/README.html#generating-a-new-ssh-key-pair) ssh e:

1. abilitare la **chiave pubblica** come [deploy key](https://docs.gitlab.com/ce/ssh/README.html#deploy-keys) del progetto `maven-repo`
2. impostare la **chiave privata** come [variabile segreta](https://gitlab.com/help/ci/variables/README#secret-variables) del progetto che contiene i sorgenti della libreria (o del gruppo che lo contiene)
3. aggiungere al file `.gitlab-ci.yml` uno script che inizializzi la chiave privata nel contenitore docker all'interno del quale viene eseguito il processo maven

```
before_script:
  # cfr. https://docs.gitlab.com/ee/ci/ssh_keys/README.html
  # Install ssh-agent if not already installed, it is required by Docker.
  # (change apt-get to yum if you use a CentOS-based image)
  - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y )'
  # alpine - 'which ssh-agent || ( apk update && apk upgrade && apk add --no-cache bash git openssh)'

  # Run ssh-agent (inside the build environment)
  - eval $(ssh-agent -s)

  # Add the SSH key stored in SSH_PRIVATE_KEY variable to the agent store
  # alpine - printf '%s\n' "$SSH_PRIVATE_KEY" | ssh-add -
  - ssh-add <(echo "$SSH_PRIVATE_KEY")

  # For Docker builds disable host key checking. Be aware that by adding that
  # you are suspectible to man-in-the-middle attacks.
  # WARNING: Use this only with the Docker executor, if you use it with shell
  # you will overwrite your user's SSH config.
  #- mkdir -p ~/.ssh
  - '[[ -f /.dockerenv ]] && mkdir -p ~/.ssh && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
  # In order to properly check the server's host key, assuming you created the
  # SSH_SERVER_HOSTKEYS variable previously, uncomment the following two lines
  # instead.
  - '[[ ! -f /.dockerenv ]] && mkdir -p /root/.ssh && echo "$SSH_PRIVATE_KEY" > /root/.ssh/id_rsa && chmod 600 /root/.ssh/id_rsa && chmod 700 /root/.ssh'
  - '[[ -f /.dockerenv ]] && mkdir -p /root/.ssh && echo "$SSH_SERVER_HOSTKEYS" > /root/.ssh/known_hosts'
```

A questo punto quando viene push-ato sul remote repository un tag si scatena il processo di compilazione, test e deploy del jar (non starò qui adesso a dire che il tutto è splendidamente guidato da [git flow](https://github.com/petervanderdoes/gitflow-avh)).

Resta solo da capire come usare la libreria testè deployata: ovviamente per usarla è sufficiente dichiararla come dipendenza all'interno del file pom del progetto che la utilizza avendo cura di indicare il riferimento al repositoy con:

```xml
 <repositories>
        <repository>
            <id>garanteasy-repo</id>
            <name>Garanteasy repository</name>
            <url>https://gitlab.com/garanteasy/maven-repo/raw/releases</url>
        </repository>
</repositories>
```

Ancora una volta, trattandosi di un progetto privato dobbiamo impostare un meccanismo di autenticazione; per come è fatto gitlab non è sufficiente impostare le credenziali come indicato nella documentazione maven all'interno del file `$HOME/.m2/settings.xml` o `$HOME/.m2/settings-security.xml`, dobbiamo bensì indicare un custom header in questa forma:

```xml
    <servers>
    ...
        <server>
            <id>garanteasy-repo</id>
            <configuration>
                <httpHeaders>
                    <property>
                        <name>PRIVATE-TOKEN</name>
                        <value>xxxxxxxxxxxxxxx</value>
                    </property>
                </httpHeaders>
            </configuration>
        </server>
    </servers>
```

NB l'id del server all'interno del file settings.xml è il medesimo id del server indicato nel pom.xml.

Quale valore dobbiamo assegnare al private token e come automatizziamo la build per il sistema di continuous integration?

Il **valore del token** altro non è che il valore di un [personal access token](https://docs.gitlab.com/ce/user/profile/personal_access_tokens.html) che puoi generare all'interno della [tua pagina di profilo](https://gitlab.com/profile/personal_access_tokens).

Per automatizzare la build, dopo avere assegnato il valore del token ad una [variabile segreta](https://gitlab.com/help/ci/variables/README#secret-variables),  puoi sfruttare lo stesso sistema usato per eseguire uno script prima della compilazione:

```yml
before_script:
  - echo '<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
                <localRepository>${CI_PROJECT_DIR}/.m2/repository</localRepository>
            <servers>
              <server>
                <id>garanteasy-repo</id>
                <configuration>
                  <httpHeaders>
                    <property>
                      <name>PRIVATE-TOKEN</name>
                      <value>${PRIVATE_TOKEN}</value>
                    </property>
                  </httpHeaders>
                </configuration>
              </server>
            </servers>
          </settings>' > $HOME/.m2/settings.xml
```
