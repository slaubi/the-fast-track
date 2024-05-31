Utiliser des branches
=====================

Il existe de nombreuses façons d'organiser le workflow des changements apportés au code d'un projet. Mais travailler directement sur la branche *master* de Git et déployer directement en production sans tester n'est probablement pas la meilleure solution.

Tester ne se résume pas à un test unitaire ou fonctionnel, il s'agit aussi de vérifier le comportement de l'application avec les données de production. Le fait que vous, ou vos `collègues`_, puissiez utiliser l'application exactement de la même manière que lorsqu'elle sera déployée est un énorme avantage. Cela vous permet de déployer en toute confiance. C'est particulièrement vrai lorsque des personnes non-techniques peuvent valider de nouvelles fonctionnalités.

Par souci de simplicité et pour éviter de nous répéter, nous continuerons à travailler sur la branche *master* de Git dans les prochaines étapes, mais voyons comment nous pourrions améliorer cela.

Adopter un workflow Git
-----------------------

Un workflow possible est de créer une branche par nouvelle fonctionnalité ou correction de bogue. C'est simple et efficace.

Créer des branches
-------------------

.. index::
    single: Git;branch
    single: Git;checkout

Le workflow commence par la création d'une branche Git :

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

Cette commande crée une branche ``sessions-in-db`` à partir de la branche ``master``. Elle "*fork*" le code et la configuration de l'infrastructure.

Stocker les sessions dans la base de données
---------------------------------------------

.. index::
    single: Session;Database

Comme vous l'avez deviné d'après le nom de la branche, nous voulons passer du stockage de session dans le système de fichiers à un stockage en base de données (notre base de données PostgreSQL ici).

Les étapes nécessaires pour le faire sont classiques :

#. Créez une branche Git ;

#. Mettez à jour la configuration de Symfony si nécessaire ;

#. Écrivez et/ou mettez à jour le code si nécessaire ;

#. Update the PHP configuration if needed (like adding the PostgreSQL PHP
   extension);

#. Mettez à jour l'infrastructure pour Docker et Platform.sh si nécessaire (ajoutez le service PostgreSQL) ;

#. Testez localement ;

#. Testez à distance ;

#. *Mergez* la branche dans master ;

#. Déployez en production ;

#. Supprimez la branche.

Pour stocker les sessions en base de données, modifiez la configuration ``session.handler_id`` pour pointer sur le DSN de la base de données :

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -9,7 +9,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(resolve:DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

Pour stocker les sessions en base de données, nous devons créer une table ``sessions``. Faites-le avec une migration Doctrine :

.. code-block:: terminal

    $ symfony console make:migration

Migrez la base de données :

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Testez localement en naviguant sur le site. Comme il n'y a pas de changement visuel et que nous n'utilisons pas encore les sessions, tout devrait continuer à fonctionner comme avant.

.. note::

    Nous n'avons pas besoin des étapes 3 à 5 ici puisque nous ré-utilisons la base de données comme stockage de session, mais le chapitre à propos de l'utilisation de Redis montre à quel point il est facile d'ajouter, tester et déployer un nouveau service avec Docker et Platform.sh.

Committez vos changements sur la nouvelle branche :

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Déployer une branche
---------------------

.. index::
    single: Platform.sh;Environment

Avant le déploiement en production, nous devrions tester la branche sur la même infrastructure que celle de production. Nous devrions également valider que tout fonctionne bien pour l'environnement ``prod`` de Symfony (le site local utilise l'environnement ``dev`` de Symfony).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Maintenant, créons un *environnement Platform.sh* basé sur la *branche Git* :

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:deploy

Cette commande crée un nouvel environnement comme suit :

* La branche hérite du code et de l'infrastructure de la branche Git actuelle (``sessions-in-db``) ;

* Les données proviennent de l'environnement ``master`` (c'est-à-dire la production) en prenant un instantané de toutes les données du service, y compris les fichiers (fichiers uploadés par l'internaute par exemple) et les bases de données ;

* Un nouveau cluster dédié est créé pour déployer le code, les données et l'infrastructure.

Comme le déploiement suit les mêmes étapes que le déploiement en production, les migrations de bases de données seront également exécutées. C'est un excellent moyen de valider que les migrations fonctionnent avec les données de production.

Les environnements autres que ``master`` sont très similaires à ``master``, à quelques petites différences près : par exemple, les emails ne sont pas envoyés par défaut.

.. index::
    single: Symfony CLI;cloud:url

Une fois le déploiement terminé, ouvrez la nouvelle branche dans un navigateur :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Notez que toutes les commandes Platform.sh fonctionnent sur la branche Git courante. Cela ouvrira l'URL de la branche ``sessions-in-db`` déployée. L'URL ressemblera à ``https://sessions-in-db-xxx.eu-5.platformsh.site/``.

Testez le site web sur ce nouvel environnement. Vous devriez voir toutes les données que vous avez créées dans l'environnement ``master``.

Si vous ajoutez d'autres conférences sur l'environnement ``master``, elles n'apparaîtront pas dans l'environnement ``sessions-in-db`` et vice-versa. Les environnements sont indépendants et isolés.

Si le code évolue sur master, vous pouvez toujours *rebaser* la branche Git et déployer la version mise à jour, résolvant ainsi les conflits tant pour le code que pour l'infrastructure.

.. index::
    single: Symfony CLI;cloud:env:sync

Vous pouvez même synchroniser les données de master avec l'environnement ``sessions-in-db`` :

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Déboguer les déploiements en production avant de déployer
------------------------------------------------------------

.. index::
    single: Platform.sh;Debugging

Par défaut, tous les environnements Platform.sh utilisent les mêmes paramètres que l'environnement ``master``/``prod`` (c'est à dire l'environnement Symfony ``prod``). Cela vous permet de tester l'application dans des conditions réelles. Il vous donne l'impression de développer et de tester directement sur des serveurs de production, mais sans les risques qui y sont associés. Cela me rappelle le bon vieux temps où nous déployions par FTP.

.. index::
    single: Symfony CLI;cloud:env:debug

En cas de problème, vous pouvez passer à l'environnement Symfony ``dev`` :

.. code-block:: terminal

    $ symfony cloud:env:debug

Une fois terminé, revenez aux réglages de production :

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    N'activez **jamais** l'environnement ``dev`` et n'activez jamais le Symfony Profiler sur la branche ``master`` ; cela rendrait votre application vraiment lente et ouvrirait de nombreuses failles de sécurité graves.

Tester les déploiements en production avant de déployer
---------------------------------------------------------

L'accès à la prochaine version du site web avec les données de production ouvre de nombreuses opportunités : des tests de régression visuelle aux tests de performance. `Blackfire`_ est l'outil parfait pour ce travail.

Reportez-vous à l'étape :doc:`Performances <29-performance>` pour en savoir plus sur la façon dont vous pouvez utiliser Blackfire pour tester votre code avant de le déployer.

Merger en production
--------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Lorsque vous êtes satisfait des changements de la branche, mergez le code et l'infrastructure dans la branche ``master`` de Git :

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

Et déployez :

.. code-block:: terminal

    $ symfony cloud:deploy

Lors du déploiement, seuls le code et les changements d'infrastructure sont poussés vers Platform.sh ; les données ne sont en aucun cas affectées.

Faire le ménage
----------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Enfin, faites le ménage en supprimant la branche Git et l'environnement Platform.sh :

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Aller plus loin

    * `Les branches Git`_ ;

.. _`collègues`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Les branches Git`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
