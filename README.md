# Introduction

L'objectif de ce guide est de configurer un environnement de formation [Ansible](https://www.ansible.com/) en utilisant des conteneurs [Docker](https://www.docker.com/). Après avoir terminé ce tutoriel, vous aurez un conteneur maître Docker qui peut gérer trois conteneurs hôtes (vous pouvez facilement étendre le nombre d'hôtes gérés pour répondre à vos besoins).

Pourquoi ai-je décidé d'utiliser Docker au lieu d'une virtualisation conventionnelle comme [VirtualBox](https://www.virtualbox.org/) ? Les conteneurs Docker consomment beaucoup moins de ressources, ce qui vous permet de créer des environnements de test plus importants sur votre ordinateur portable. Les conteneurs Docker sont beaucoup plus rapides à démarrer et à arrêter que les machines virtuelles standard, ce qui est important lorsque vous expérimentez et que vous montez et descendez tout l'environnement. J'ai utilisé [Docker Compose] (https://docs.docker.com/compose/overview/) pour automatiser la configuration de l'environnement de laboratoire (il n'est pas nécessaire de maintenir chaque conteneur séparément).

Ce guide **n'est pas** un tutoriel Ansible ou Docker (bien que j'explique certains concepts de base). Son but est uniquement de mettre en place un environnement de laboratoire pour permettre des expériences avec Ansible sur une machine locale.

**IMPORTANT** : Afin de suivre ce tutoriel, vous devez installer Docker CE (Community Edition) sur votre machine. L'installation est bien documentée sur https://docs.docker.com/engine/installation/#supported-platforms et je ne la couvrirai pas ici.

Une brève description d'Ansible et de Docker :

## Ansible

Ansible est un système d'automatisation informatique. Il gère la gestion de la configuration, le déploiement d'applications, l'approvisionnement du cloud, l'exécution de tâches ad hoc et l'orchestration multinode - y compris la banalisation de choses comme les mises à jour sans temps d'arrêt avec des équilibreurs de charge.

Vous pouvez en savoir plus à l'adresse suivante : [www.ansible.com](https://www.ansible.com/)

## Docker

Docker est la première plate-forme logicielle de conteneurs au monde. Les développeurs utilisent Docker pour éliminer les problèmes de "fonctionnement sur ma machine" lorsqu'ils collaborent sur du code avec leurs collègues. Les opérateurs utilisent Docker pour exécuter et gérer des applications côte à côte dans des conteneurs isolés afin d'obtenir une meilleure densité de calcul. Les entreprises utilisent Docker pour créer des pipelines de livraison de logiciels agiles afin d'expédier de nouvelles fonctionnalités plus rapidement, en toute sécurité et en toute confiance pour les applications Linux, Windows Server et Linux-on-mainframe. 

Vous pouvez en savoir plus à l'adresse suivante : [www.docker.com](https://www.docker.com/)

# Démarrage rapide

## Cloner le dépôt

Clonez ce dépôt git :

`git clone https://github.com/LMtx/ansible-lab-docker.git`

## Construire des images et exécuter des conteneurs

Entrez dans le répertoire **ansible** contenant le fichier [docker-compose.yml](./ansible/docker-compose.yml).

Construire des images docker et exécuter des conteneurs en arrière-plan (détails définis dans [docker-compose.yml](./ansible/docker-compose.yml)) :

`docker-compose up -d --build``.

Connectez-vous au **nœud principal** :

`docker exec -it master01 bash`

Vérifier si la connexion réseau fonctionne entre les hôtes maître et gérés :

`ping -c 2 host01`

Démarrez un [Agent SSH] (https://man.openbsd.org/ssh-agent) sur le **nœud principal** pour gérer les clés SSH protégées par une phrase de passe :

`ssh-agent bash`

Chargez la clé privée dans l'agent SSH afin de permettre l'établissement de connexions sans avoir à saisir la phrase de passe de la clé à chaque fois :

`ssh-add master_key`

    Entrez la phrase de passe pour la clé maître :

La **phrase de passe** par défaut est : `12345`

La phrase de passe de la clé par défaut a pu être modifiée dans [ansible/master/Dockerfile](./ansible/master/Dockerfile).

## Playbooks Ansible

Exécutez un [sample ansible playbook](./ansible/master/ansible/ping_all.yml) qui vérifie la connexion entre le noeud maître et les hôtes gérés :

`ansible-playbook -i inventory ping_all.yml`

Confirmez _chaque_ nouvel hôte pour les connexions SSH :

    L'empreinte de la clé ECDSA est SHA256:HwEUUnBtOm9hVAR2PJflNdCVchSCzIlpOpqYlwp+w+w.
    Êtes-vous sûr de vouloir continuer à vous connecter (oui/non) ?

Tapez : `yes` (trois fois)

Installer PHP sur le web **groupe d'inventaire** :

Afin de regrouper les hôtes gérés pour faciliter la maintenance, vous pouvez utiliser des groupes dans le [fichier d'inventaire] ansible (./ansible/master/ansible/inventory).

Exécutez un [sample ansible playbook](./ansible/master/ansible/install_php.yml) :

`ansible-playbook -i inventaire install_php.yml`

## Copier les données entre le système de fichiers local et les conteneurs

### Copier le répertoire du conteneur vers le système de fichiers local

`docker cp master01:/var/ans/ .`

### Copier le répertoire du système de fichiers local vers le conteneur :

`docker cp ./ans master01:/var/`

Vous pouvez vérifier l'utilisation en exécutant :

`docker cp --help`

## Nettoyage

Lorsque vous avez terminé vos expériences ou que vous voulez détruire l'environnement du laboratoire pour en apporter un nouveau, exécutez les commandes suivantes.

Arrêtez les conteneurs :

`docker-compose kill`

Supprimez les conteneurs :

`docker-compose rm`


Supprimez le volume :

`docker volume rm ansible_ansible_vol`

Si vous voulez, vous pouvez supprimer les images Docker (bien que cela ne soit pas nécessaire pour démarrer un nouvel environnement de laboratoire) :

`docker rmi ansible_host ansible_master ansible_base`

# Conseils

Afin de partager la clé SSH publique entre les conteneurs **master** et **host** j'ai utilisé le **volume** de Docker monté sur tous les conteneurs :

[docker-compose.yml](./ansible/docker-compose.yml) :

    [...]
    volumes :
      - ansible_vol:/var/ans
    [...]

Le conteneur maître stocke la clé SSH dans ce volume ([ansible/master/Dockerfile](./ansible/master/Dockerfile)) :

    [...]
    WORKDIR /var/ans
    RUN ssh-keygen -t rsa -N 12345 -C "master key" -f master_key
    [...]

Et les conteneurs hôtes ajoutent la clé publique SSH au fichier authorized_keys ([ansible/host/run.sh](./ansible/host/run.sh)) afin d'autoriser les connexions depuis le maître :

    cat /var/ans/master_key.pub >> /root/.ssh/authorized_keys

**IMPORTANT:** cette configuration est valable pour un environnement de laboratoire mais pour un déploiement en production vous devez distribuer la clé publique d'une autre manière.

## Dépannage

## Les conteneurs hôtes s'arrêtent après leur création

Vérifiez que le type de fin de ligne de [ansible/hosts/run.sh](./ansible/host/run.sh) est correct - il **doit être Linux/Unix (LF)** et non Windows (CRLF). Vous pouvez changer le type de fin de ligne en utilisant un éditeur de code source (comme Notepad++ ou Visual Studio Code) ; sous Linux vous pouvez utiliser la commande `dos2unix`.

## Autre problème

Veuillez ouvrir un [problème] (https://github.com/LMtx/ansible-lab-docker/issues/new) et j'essaierai de vous aider.
