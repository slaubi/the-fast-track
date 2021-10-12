Votre environnement de travail
==============================

Avant de commencer à travailler sur le projet, nous devons nous assurer que tout le monde a un environnement de travail de qualité. C'est très important. Les outils de développement dont nous disposons aujourd'hui sont très différents de ceux que nous avions il y a 10 ans. Ils ont beaucoup évolué et en bien. Il serait donc dommage de ne pas les exploiter : de bons outils vous mèneront loin.

S'il vous plaît, ne sautez pas cette étape. Ou du moins, prenez connaissance de la dernière partie à propos de la commande ``symfony`` (partie *Symfony CLI*).

Un ordinateur
-------------

Vous avez besoin d'un ordinateur. La bonne nouvelle c'est qu'il peut fonctionner avec la plupart des systèmes d'exploitation : macOS, Windows, ou Linux. Symfony et tous les outils que nous allons utiliser sont compatibles avec chacun d'entre eux.

Choix arbitraires
-----------------

Je veux pouvoir avancer rapidement avec les meilleures options possibles. J'ai donc fait des choix arbitraires pour ce livre.

Nous utiliserons `PostgreSQL`_ pour tout : du moteur de base de données aux files d'attente, et du cache au stockage de session. Pour la plupart des projets, PostgreSQL est la meilleure solution, puisqu'elle s'adapte bien et permet de simplifier l'architecture en n'ayant qu'un seul service à gérer.

A la fin du livre, nous apprendrons comment utiliser `RabbitMQ`_ pour les files d'attente (*queues*) et `Redis`_ pour les sessions.

IDE
---

.. index:: IDE

Vous pouvez utiliser Notepad si vous le souhaitez. Cependant, je ne le recommanderais pas.

J'ai travaillé avec Textmate, mais je ne l'utilise plus aujourd'hui. Le confort d'utilisation d'un "vrai" IDE n'a pas de prix. L'auto-complétion, l'ajout et le tri automatique des ``use``, le passage rapide d'un fichier à l'autre... Autant de fonctionnalités qui vont largement augmenter votre productivité.

Je recommande d'utiliser `Visual Studio Code`_ ou `PhpStorm`_. Le premier est gratuit et le second ne l'est pas, mais PhpStorm offre une meilleure intégration avec Symfony (grâce au `Symfony Support Plugin`_). Au final, c'est votre choix. Je sais que vous voulez savoir quel IDE j'utilise. J'écris ce livre avec Visual Studio Code.

Terminal
--------

.. index:: Terminal

Nous passerons de l'IDE à la ligne de commande en permanence. Vous pouvez utiliser le terminal intégré à votre IDE, mais je préfère utiliser un terminal indépendant pour avoir plus d'espace.

Linux est fourni avec ``Terminal``. Utilisez `iTerm2`_ sous macOS. Sous Windows, `Hyper`_ fonctionne bien.

Git
---

.. index:: Git

Mon dernier livre recommandait Subversion pour la gestion de versions, mais on dirait bien que maintenant tout le monde utilise `Git`_.

Sur Windows, installez `Git bash`_.

Assurez-vous de connaître les commandes de base comme ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``, etc.

PHP
---

.. index:: PHP

Nous utiliserons Docker pour les services, mais j'aime avoir PHP installé sur mon ordinateur local pour des raisons de performance, de stabilité et de simplicité. C'est peut-être vieux jeu, mais la combinaison d'un PHP local et des services Docker est parfaite pour moi.

Utilisez PHP 8.0 et vérifiez que les extensions PHP suivantes soient installées, ou installez-les maintenant : `intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Éventuellement, vous pouvez également installer ``redis``, ``curl`` et ``zip``.

Vous pouvez vérifier les extensions actuellement activées avec ``php -m``.

Nous avons aussi besoin de ``php-fpm`` s'il est disponible sur votre plate-forme, mais ``php-cgi`` peut être une alternative.

Composer
--------

.. index:: Composer

Gérer les dépendances est capital aujourd'hui sur un projet Symfony. Installez la dernière version de `Composer`_, le gestionnaire de paquets pour PHP.

Si vous ne connaissez pas Composer, prenez le temps de vous familiariser avec cet outil.

.. tip::

    Vous n'avez pas besoin de taper le nom complet des commandes : ``composer req`` fait la même chose que ``composer require``, utilisez ``composer rem`` au lieu de ``composer remove``, etc.

NodeJS
------

Nous n'écrirons pas beaucoup de code JavaScript, mais nous utiliserons les outils JavaScript/NodeJS pour gérer nos assets. Vérifiez que vous avez installé `NodeJS`_ et le gestionnaire de paquets `Yarn`_.

Docker et Docker Compose
------------------------

.. index:: Docker,Docker Compose

Les services seront gérés par Docker et Docker Compose. `Installez-les`_ et lancez Docker. Si vous utilisez cet outil pour la première fois, familiarisez-vous avec lui. Mais ne paniquez pas, notre utilisation de Docker sera très simple : pas de configuration alambiquée, ni de réglage complexe.

Symfony CLI
-----------

.. index:: Symfony CLI

Finalement, nous utiliserons la commande ``symfony`` pour accroître notre productivité. Du serveur web local qu'il fournit à l'intégration complète de Docker, en passant par le support de SymfonyCloud, nous gagnerons un temps considérable.

Installez la `commande symfony`_  et placez-la dans votre ``$PATH``. Créez un compte `SymfonyConnect`_  si vous n'en avez pas déjà un et connectez-vous avec la commande ``symfony login``.

Pour pouvoir utiliser HTTPS localement, nous avons également besoin d'`installer une autorité de certification`_ pour activer le support TLS. Exécutez la commande suivante :

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Vérifiez que votre ordinateur répond aux conditions requises en exécutant la commande suivante :

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

Si vous voulez quelque chose de plus joli, vous pouvez également utiliser le `proxy Symfony`_. C'est optionnel, mais ce proxy vous permet d'obtenir un nom de domaine local se terminant par ``.wip`` pour votre projet.

Lorsque nous exécuterons des commandes dans le terminal, nous les préfixerons presque toujours avec ``symfony``, comme dans ``symfony composer`` au lieu de simplement ``composer``, ou ``symfony console`` au lieu de ``./bin/console``.

La raison principale est que la commande ``symfony`` définit automatiquement certaines variables d'environnement à partir des services exécutés via Docker. Ces variables d'environnement sont injectées par le serveur web local et disponibles automatiquement pour les requêtes HTTP. L'utilisation de ``symfony`` dans l'invite de commande vous assure donc d'obtenir le même comportement partout.

De plus, la commande ``symfony`` sélectionne automatiquement la "meilleure" version de PHP possible pour le projet.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`NodeJS`: https://nodejs.org/
.. _`Yarn`: https://classic.yarnpkg.com/en/docs/install/
.. _`Install them`: https://docs.docker.com/install/
.. _`Symfony CLI`: https://symfony.com/download
.. _`SymfonyConnect`: https://symfony.com/connect/login
.. _`install a certificate authority (CA)`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`Symfony proxy`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
