---
title: "Pihole et les conteneurs Docker"
description: "Résoudre un problème lorsqu’on utilise Pihole avec des conteneurs."
date: 2020-06-15
layout: post
keywords: privacy raspberry pihole docker
categories: tech
language: fr_FR
---

J’ai récemment rencontré un problème en combinant l’utilisation de
[Pihole](https://pi-hole.net/) et de conteneurs
[Docker](https://www.docker.com/). Je vais détailler ce qui se passe et donner
une manière de résoudre ça.

## Contexte et problème

```
   ┌──────────────────────── Hôte ────────────────────────┐
   │                                                      │
   │                                                      │
   │    ┌─────────────────────────┐                       │
   │    │                         │                       │
   │    │                      ┌──┴───┐                ┌──┴───┐
   │    │                      │ Ports◄────────────────► Ports│
   │    │                      │ DNS  │                │ DNS  │
   │    │  Conteneur Pihole    └──┬───┘                └──┬───┘
   │    │                         │                       │
   │    │                         ├───────┐               │
   │    └─────────────────────────┘       │               │
   │                                      │               │
   │    ┌────────────────────┐            │               │
   │    │  Conteneur Docker  ├────────────┤               │
   │    └────────────────────┘            │               │
   │                                   Réseau             │
   │                                   Docker             │
   │    ┌────────────────────┐            │               │
   │    │  Conteneur Docker  ├────────────┘               │
   │    └────────────────────┘                            │
   │                                                      │
   └──────────────────────────────────────────────────────┘
```

Il s’agit d’un cas de figure où plusieurs conteneurs tournent sur un seul hôte,
et l’un de ces conteneurs est le serveur Pihole. Ce serveur DNS est utilisé par
d'autres machines, l’hôte lui-même, et les autres conteneurs.

Ce qui se passe si on laisse une configuration par défaut, c’est que
**les autres conteneurs n’arrivent pas à effectuer de requêtes DNS**. Je ne
sais pas l’expliquer, il faudrait faire une analyse de paquets pour voir ce
qu’il se passe réellement. Quoi qu’il en soit moi ça me pose problème par
exemple parce que l’un de mes conteneurs est un serveur
[Nextcloud](https://nextcloud.com/) qui fait des requêtes vers l’extérieur.

Sur ce sujet, j’ai trouvé les ressources suivantes :

- Une publication [Reddit](https://www.reddit.com/r/pihole/comments/e4y63v/whats_the_correct_way_to_use_pihole_in_docker/).
- Une [issue](https://github.com/pi-hole/docker-pi-hole/issues/438) sur le
  Github de Pihole.
- Un post [Stackoverflow](https://stackoverflow.com/questions/64007727/docker-compose-internal-dns-server-127-0-0-11-connection-refused)

## Solution

Une solution est d’héberger Pihole sur un autre hôte. Mais moi je n’ai qu’une
seule machine pour tout ça alors non.

C’est pas idéale, mais il faut renseigner l’adresse IP du DNS à chaque
conteneur qui en a besoin. Cela implique déjà de donner une adresse statique à
Pihole. Moi j’utilise [docker-compose](https://docs.docker.com/compose/) pour
tous mes conteneurs alors je vais vous donner de la configuration pour ça.

1. Connecter tous les conteneurs sur un même réseau docker (il y a plusieurs
   solutions, moi j’ai créé un réseau manuellement en ligne de commandes). Dans
   chaque `docker-compose.yml` je déclare ce réseau externe et j’indique aux
   conteneurs de l’utiliser :
   ```yaml
   version: '3'
   networks:
     my_network:
       external:
         name: my-external-network
   services:
     my_app:
       # d’autres lignes que je ne mets pas ici…
       networks:
         - my_network
   ```
2. Mais ça tout seul, ça ne fonctionne pas (comme dit plus haut). Il faut déjà
   donner une adresse statique à Pihole (votre adresse dépend de votre réseau,
   moi j’ai un réseau en `172.23.0.0/16`. Voici donc le morceau de
   configuration modifié :
   ```yaml
   services:
     pihole:
       networks:
         my_network:
           ipv4_address: 172.23.0.200
   ```
3. Ensuite, il faut indiquer aux autres conteneurs d’utiliser ce DNS :
   ```yaml
   # Bien sûr je ne vous remets pas tout le fichier
   services:
     my_app:
       dns: 172.23.0.200
   ```

Et voilà, les conteneurs utiliseront directement l’adresse du conteneur Pihole
pour leurs requêtes DNS.

C’est clairement moche cette adresse IP statique. Si vous avez des suggestions
pour rendre ça plus propre, je suis preneur !
