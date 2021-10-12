Présentation du projet
=======================

Nous devons trouver un projet sur lequel travailler. C'est un certain défi car nous devons choisir un projet assez vaste pour couvrir complètement Symfony, mais en même temps, il devrait être assez petit ; je ne veux pas que vous vous ennuyiez à implémenter des fonctionnalités similaires plusieurs fois.

Description du projet
---------------------

Comme le livre doit sortir pendant la SymfonyCon d'Amsterdam, ce serait intéressant si le projet était en quelque sorte relié à Symfony et aux conférences. Et pourquoi pas un `livre d'or <https://en.wikipedia.org/wiki/Guestbook>`_ ? J'aime le côté démodé et désuet de développer un livre d'or en 2019 !

Nous tenons notre sujet. Le projet a pour but d'obtenir un retour d'expérience sur les conférences : une liste des conférences sur la page d'accueil ainsi qu'une page pour chacune d'entre elles, pleine de commentaires sympathiques. Un commentaire est composé d'un petit texte et d'une photo, optionnelle, prise pendant la conférence. Je suppose que je viens de rédiger toutes les spécifications dont nous avons besoin pour commencer.

Le *projet* comprendra plusieurs *applications* : une application web traditionnelle avec une interface HTML, une API et une SPA pour les téléphones mobiles. Qu'en dites-vous ?

La maîtrise s’acquiert par la pratique
-----------------------------------------

La maîtrise s’acquiert par la pratique. Point final. Lire un livre sur Symfony, c'est bien. Coder une application sur votre ordinateur tout en lisant un livre sur Symfony, c'est encore mieux. Ce livre est très spécial puisque tout a été fait pour que vous puissiez suivre, coder, et obtenir les mêmes résultats que ceux que j'avais localement sur ma machine lorsque je l'ai fait.

Le livre contient tout le code que vous devez écrire ainsi que toutes les commandes que vous devez exécuter pour arriver au résultat final. Il ne manque aucun code. Toutes les commandes sont présentes. C'est possible parce que les applications développées avec Symfony n'ont besoin que de très peu de code pour démarrer. La plupart du code que nous allons écrire ensemble concerne la *logique métier* du projet. Tout le reste est automatisé ou généré automatiquement pour nous.

À propos du diagramme de l'infrastructure finale
-------------------------------------------------

Même si l'idée du projet semble simple, nous n'allons pas construire un projet de type "Hello World". Nous n'utiliserons pas seulement PHP et une base de données.

Le but est de créer un projet avec des complexités que vous pourriez retrouver dans la vie réelle. Vous voulez une preuve ? Jetez un coup d'œil à l'infrastructure finale du projet :

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

L'un des avantages majeurs d'utiliser un framework est la faible quantité de code nécessaire pour développer un tel projet :

* 20 classes PHP sous ``src/`` pour le site ;

* 550 lignes logiques de code (LLOC) de PHP, tel que rapporté par `PHPLOC <https://github.com/sebastianbergmann/phploc>`_ ;

* 40 lignes de configuration personnalisée réparties dans 3 fichiers (via annotations et YAML), principalement pour configurer le design de l'interface d'administration ;

* 20 lignes de configuration de l'infrastructure de développement (Docker) ;

* 100 lignes de configuration de l'infrastructure de production (SymfonyCloud) ;

* 5 variables d'environnement explicites.

Envie de relever le défi ?

Récupérer le code source du projet
------------------------------------

Pour continuer sur le thème à l'ancienne, j'aurais pu créer un CD contenant le code source, non ? Mais que diriez-vous d'un dépôt Git à la place ?

.. index::
    single: Project;Git Repository
    single: Git;clone

Clonez le `dépôt du livre d'or <https://github.com/the-fast-track/book-5.2-2>`_ quelque part sur votre machine :

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.2-2 --book guestbook

Ce dépôt contient tout le code source du livre.

Notez que nous utilisons ``symfony new`` au lieu de ``git clone`` puisque la commande fait bien plus que simplement cloner le dépôt (hébergé sur Github dans l'organisation ``the-fast-track`` : ``https://github.com/the-fast-track/book-5.2-2``). Elle démarre également le serveur web et les conteneurs, migre la base de données, charge les données de test, etc. Après l'exécution de la commande, le site devrait être opérationnel, prêt à être utilisé.

Le code source est synchronisé à 100% avec le code source du livre (utilisez l'URL exacte du dépôt, indiquée ci-dessus). Essayer de synchroniser manuellement les changements du livre avec le code source du dépôt est presque impossible. J'ai tenté de le faire et je n'y suis pas arrivé. C'est tout simplement impossible. Surtout pour des livres comme ceux que j'écris, qui vous racontent l'histoire du développement d'un site web. Comme chaque chapitre dépend des précédents, un changement sur l'un d'eux peut avoir des conséquences sur tous les suivants.

La bonne nouvelle est que le dépôt Git pour ce livre est *généré automatiquement* à partir du contenu du livre. Pas mal, non ? J'aime tout automatiser, par conséquent il y a un script dont le travail est de lire le livre et de créer le dépôt Git. Il y a un effet secondaire sympa : lors de la mise à jour du livre, le script échouera si les changements sont incohérents ou si j'oublie de mettre à jour certaines instructions. C'est du BDD, *Book Driven Development* !

Parcourir le code source
------------------------

Mieux encore, le dépôt ne contient pas seulement la version finale du code source sur la branche ``master`` : le script exécute chaque action expliquée dans le livre, puis *commit* son travail à la fin de chaque section. Il *tag* également chaque étape et sous-étape pour faciliter la navigation dans le code. Joli, n'est-ce pas ?

.. index::
    single: Git;checkout

Si vous êtes du genre à prendre des raccourcis, vous pouvez récupérer le code source correspondant à la fin d'une étape du livre grâce à son *tag*. Par exemple, si vous souhaitez lire et tester le code à la fin de l'étape 10, procédez comme suit :

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Comme pour le clonage du dépôt, nous n'utilisons pas ``git checkout`` mais plutôt ``symfony book:checkout``. Cette commande s'assure que, quel que soit l'état dans lequel votre code se trouve actuellement, vous obteniez un site web fonctionnel pour l'étape que vous demandez. **Faites attention : toutes les données, le code source et les conteneurs sont supprimés par cette opération.**

Vous pouvez également récupérer n'importe quelle sous-étape :

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

Encore une fois, je vous recommande fortement de coder par vous-même. Mais si vous avez un problème, vous pouvez toujours comparer ce que vous avez avec le contenu du livre.

.. index::
    single: Git;diff

Vous avez un doute sur l'étape 10.2 ? Récupérez le *diff* :

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Vous voulez savoir quand un fichier a été créé ou modifié ?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

Vous pouvez également parcourir les *diffs*, les *tags* et les *commits* directement sur GitHub. C'est une excellente façon de copier/coller du code si vous lisez un livre papier !
