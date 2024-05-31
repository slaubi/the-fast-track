Mettre en place une base de données
====================================

.. index::
    single: Database

Le site web du livre d'or de la conférence permet de recueillir des commentaires pendant les conférences. Nous avons besoin de stocker ces commentaires dans un stockage persistant.

Un commentaire est mieux décrit par une structure de données fixe : un nom, un email, le texte du commentaire et une photo facultative. Ce type de données se stocke facilement dans un moteur de base de données relationnelle traditionnel.

PostgreSQL est le moteur de base de données que nous allons utiliser.

Ajouter PostgreSQL à Docker Compose
------------------------------------

.. index::
    single: Docker;PostgreSQL

Sur notre machine locale, nous avons décidé d'utiliser Docker pour gérer nos services. Le fichier ``compose.yaml`` généré contient déjà PostgreSQL en tant que service :

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

Un serveur PostgreSQL sera alors installé et certaines variables d'environnement, qui contrôlent le nom de la base de données et ses identifiants, seront configurées. Les valeurs n'ont pas vraiment d'importance.

Nous exposons également le port PostgreSQL (``5432``) du conteneur à l'hôte local. Cela nous aidera à accéder à la base de données à partir de notre machine :

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    L'extension ``pdo_pgsql`` a déjà dû être installée précédemment lors de l'installation de PHP.

Démarrer Docker Compose
------------------------

Lancez Docker Compose en arrière-plan (``-d``) :

.. code-block:: terminal

    $ docker compose up -d

Attendez un peu pour laisser démarrer la base de données, puis vérifiez que tout fonctionne bien :

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

S'il n'y a pas de conteneurs en cours d'exécution ou si la colonne ``State`` n'indique pas ``Up``, vérifiez les logs de Docker Compose :

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

Accéder à la base de données locale
--------------------------------------

L'utilitaire en ligne de commande ``psql`` peut parfois s'avérer utile. Mais vous devez vous rappelez des informations d'identification et du nom de la base de données. Encore moins évident, vous devez aussi connaître le port local sur lequel la base de données tourne sur l'hôte. Docker choisit un port aléatoire pour que vous puissiez travailler sur plus d'un projet en utilisant PostgreSQL en même temps (le port local fait partie de la sortie de ``docker compose ps``).

Si vous utilisez ``psql`` avec la commande ``symfony``, vous n'avez pas besoin de vous souvenir de quoi que ce soit.

La commande ``symfony`` détecte automatiquement les services Docker en cours d'exécution pour le projet et expose les variables d'environnement dont ``psql`` a besoin pour se connecter à la base de données.

.. index::
    single: Symfony CLI;run psql

Grâce à ces conventions, accéder à la base de données avec ``symfony run`` est beaucoup plus facile :

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Si vous n'avez pas le binaire ``psql`` sur votre hôte local, vous pouvez également l'exécuter avec ``docker compose`` :

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

Sauvegarder et restaurer les données de la base
------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Utilisez ``pg_dump`` pour sauvegarder les données de la base :

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

Et restaurer les données :

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Ajouter PostgreSQL à Platform.sh
---------------------------------

.. index::
    single: Platform.sh;PostgreSQL

Pour l'infrastructure de production sur Platform.sh, l'ajout d'un service comme PostgreSQL doit se faire dans le fichier ``.platform/services.yaml``, ce qui a déjà été fait par la recette du paquet ``webapp`` :

.. code-block:: yaml
    :caption: .platform/services.yaml
    :class: ignore

    database:
        type: postgresql:16
        disk: 1024

Le service ``database`` est une base de données PostgreSQL (même version que pour Docker) que nous voulons provisionner avec 1 Go d'espace disque.

Nous devons également "lier" la BDD au conteneur de l'application, qui est décrit dans ``.platform.app.yaml`` :

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Le service ``database`` de type ``postgresql`` est référencé comme ``database`` sur le conteneur d'application.

Vérifiez que l'extension ``pdo_pgsql`` est bien installée pour PHP :

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Accéder à la base de données de Platform.sh
----------------------------------------------

PostgreSQL fonctionne maintenant localement via Docker et en production sur Platform.sh.

Comme nous venons de le voir, exécuter``symfony run psql`` permet de se connecter automatiquement à la base de données hébergée avec Docker grâce aux variables d'environnement exposées par ``symfony run``.

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Si vous souhaitez vous connecter au PostgreSQL hébergé dans les conteneurs de production, vous pouvez ouvrir un tunnel SSH entre la machine locale et l'infrastructure Platform.sh :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Par défaut, les services Platform.sh ne sont pas exposés comme variables d'environnement sur la machine locale. Vous devez le faire explicitement en utilisant l'option ``var:expose-from-tunnel``. Pourquoi ? La connexion à la base de données de production est une opération dangereuse et vous risquez de compromettre de *vraies* données.

Comme précédemment, connectez-vous à la base de données PostgreSQL distante avec ``symfony run psql`` :

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Quand vous avez terminé, n'oubliez pas de fermer le tunnel :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Au lieu d'ouvrir un shell, vous pouvez également utiliser la commande ``symfony sql`` pour exécuter des requêtes SQL sur la base de données de production.

Exposer des variables d'environnement
-------------------------------------

.. index::
    single: Platform.sh;Environment Variables
    single: Symfony CLI;var:export

Docker Compose et Platform.sh fonctionnent en harmonie avec Symfony grâce aux variables d'environnement.

Vérifier toutes les variables d'environnement exposées par ``symfony`` en exécutant ``symfony var:export`` :

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Les variables d'environnement commençant par ``PG`` sont utilisées par l'utilitaire ``psql``. Et les autres ?

Lorsqu'un tunnel est ouvert vers Platform.sh avec l'option ``--expose-env-vars``, la commande ``var:export`` retourne les variables d'environnement distantes :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Décrire votre infrastructure
-----------------------------

Vous ne vous en êtes peut-être pas encore rendu compte, mais le fait que l'infrastructure soit stockée dans des fichiers à côté du code aide beaucoup. Docker et Platform.sh utilisent des fichiers de configuration pour décrire l'infrastructure du projet. Lorsqu'une nouvelle fonctionnalité nécessite un service supplémentaire, le code et les modifications de l'infrastructure font partie du même patch.

.. sidebar:: Aller plus loin

    * `Services Platform.sh`_ ;

    * `Tunnel Platform.sh`_ ;

    * `Documentation PostgreSQL`_ ;

    * `Les commandes docker compose`_.

.. _`Services Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Tunnel Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Documentation PostgreSQL`: https://www.postgresql.org/docs/
.. _`Les commandes docker compose`: https://docs.docker.com/compose/reference/
