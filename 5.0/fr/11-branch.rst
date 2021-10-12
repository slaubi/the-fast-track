Utiliser des branches
=====================

Il existe de nombreuses façons d'organiser le workflow des changements apportés au code d'un projet. Mais travailler directement sur la branche *master* de Git et déployer directement en production sans tester n'est probablement pas la meilleure solution.

Tester ne se résume pas à un test unitaire ou fonctionnel, il s'agit aussi de vérifier le comportement de l'application avec les données de production. Le fait que vous, ou vos `collègues`_, puissiez utiliser l'application exactement de la même manière que lorsqu'elle sera déployée est un énorme avantage. Cela vous permet de déployer en toute confiance. C'est particulièrement vrai lorsque des personnes non-techniques peuvent valider de nouvelles fonctionnalités.

Par souci de simplicité et pour éviter de nous répéter, nous continuerons à travailler sur la branche *master* de Git dans les prochaines étapes, mais voyons comment nous pourrions améliorer cela.

Adopter un workflow Git
-----------------------

Un workflow possible est de créer une branche par nouvelle fonctionnalité ou correction de bogue. C'est simple et efficace.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Créer des branches
-------------------

.. index::
    single: Git;branch
    single: Git;checkout

Le workflow commence par la création d'une branche Git :

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Cette commande crée une branche ``sessions-in-redis`` à partir de la branche ``master``. Elle "*fork*" le code et la configuration de l'infrastructure.

Stocker les sessions dans Redis
-------------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Comme vous l'avez deviné d'après le nom de la branche, nous voulons passer du stockage de session dans le système de fichiers à un stockage Redis.

Les étapes nécessaires pour le faire sont classiques :

#. Créez une branche Git ;

#. Mettez à jour la configuration de Symfony si nécessaire ;

#. Écrivez et/ou mettez à jour le code si nécessaire ;

#. Mettez à jour la configuration PHP (ajoutez l'extension Redis PHP) ;

#. Mettez à jour l'infrastructure sur Docker et SymfonyCloud (ajoutez le service Redis) ;

#. Testez localement ;

#. Testez à distance ;

#. *Mergez* la branche dans master ;

#. Déployez en production ;

#. Supprimez la branche.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Je vous laisse tester localement en naviguant sur le site. Comme il n'y a pas de changement visuel et que nous n'utilisons pas encore les sessions, tout devrait continuer à fonctionner comme avant.

Déployer une branche
---------------------

.. index::
    single: SymfonyCloud;Environment

Avant le déploiement en production, nous devrions tester la branche sur la même infrastructure que celle de production. Nous devrions également valider que tout fonctionne bien pour l'environnement ``prod`` de Symfony (le site local utilise l'environnement ``dev`` de Symfony).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Maintenant, créons un *environnement SymfonyCloud* basé sur la *branche Git* :

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Cette commande crée un nouvel environnement comme suit :

* La branche hérite du code et de l'infrastructure de la branche Git actuelle (``sessions-in-redis``) ;

* Les données proviennent de l'environnement ``master`` (c'est-à-dire la production) en prenant un instantané de toutes les données du service, y compris les fichiers (fichiers uploadés par l'internaute par exemple) et les bases de données ;

* Un nouveau cluster dédié est créé pour déployer le code, les données et l'infrastructure.

Comme le déploiement suit les mêmes étapes que le déploiement en production, les migrations de bases de données seront également exécutées. C'est un excellent moyen de valider que les migrations fonctionnent avec les données de production.

Les environnements autres que ``master`` sont très similaires à ``master``, à quelques petites différences près : par exemple, les emails ne sont pas envoyés par défaut.

.. index::
    single: Symfony CLI;open:remote

Une fois le déploiement terminé, ouvrez la nouvelle branche dans un navigateur :

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Notez que toutes les commandes SymfonyCloud fonctionnent sur la branche Git courante. Cela ouvrira l'URL de la branche ``sessions-in-redis`` déployée. L'URL ressemblera à ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Testez le site web sur ce nouvel environnement. Vous devriez voir toutes les données que vous avez créées dans l'environnement ``master``.

Si vous ajoutez d'autres conférences sur l'environnement ``master``, elles n'apparaîtront pas dans l'environnement ``sessions-in-redis`` et vice-versa. Les environnements sont indépendants et isolés.

Si le code évolue sur master, vous pouvez toujours *rebaser* la branche Git et déployer la version mise à jour, résolvant ainsi les conflits tant pour le code que pour l'infrastructure.

.. index::
    single: Symfony CLI;env:sync

Vous pouvez même synchroniser les données de master avec l'environnement ``sessions-in-redis`` :

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Déboguer les déploiements en production avant de déployer
------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Par défaut, tous les environnements SymfonyCloud utilisent les mêmes paramètres que l'environnement ``master``/``prod`` (c'est à dire l'environnement Symfony ``prod``). Cela vous permet de tester l'application dans des conditions réelles. Il vous donne l'impression de développer et de tester directement sur des serveurs de production, mais sans les risques qui y sont associés. Cela me rappelle le bon vieux temps où nous déployions par FTP.

.. index::
    single: Symfony CLI;env:debug

En cas de problème, vous pouvez passer à l'environnement Symfony ``dev`` :

.. code-block:: bash

    $ symfony env:debug

Une fois terminé, revenez aux réglages de production :

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    N'activez **jamais** l'environnement ``dev`` et n'activez jamais le Symfony Profiler sur la branche ``master`` ; cela rendrait votre application vraiment lente et ouvrirait de nombreuses failles de sécurité graves.

Tester les déploiements en production avant de déployer
---------------------------------------------------------

L'accès à la prochaine version du site web avec les données de production ouvre de nombreuses opportunités : des tests de régression visuelle aux tests de performance. `Blackfire <https://blackfire.io>`_ est l'outil parfait pour ce travail.

Reportez-vous à l'étape "Performances" pour en savoir plus sur la façon dont vous pouvez utiliser Blackfire pour tester votre code avant de le déployer.

Merger en production
--------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Lorsque vous êtes satisfait des changements de la branche, mergez le code et l'infrastructure dans la branche ``master`` de Git :

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

Et déployez :

.. code-block:: bash

    $ symfony deploy

Lors du déploiement, seuls le code et les changements d'infrastructure sont poussés vers SymfonyCloud ; les données ne sont en aucun cas affectées.

Faire le ménage
----------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Enfin, faites le ménage en supprimant la branche Git et l'environnement SymfonyCloud :

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Aller plus loin

    * `Les branches Git <https://www.git-scm.com/book/fr/v2/Les-branches-avec-Git-Les-branches-en-bref>`_ ;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`collègues`: https://en.wikipedia.org/wiki/Project_stakeholder
