Setting up a Database
=====================

.. index::
    single: Database

The Conference Guestbook website is about gathering feedback during conferences. We need to store the comments contributed by the conference attendees in a permanent storage.

A comment is best described by a fixed data structure: an author, their email, the text of the feedback, and an optional photo. The kind of data that can be best stored in a traditional relational database engine.

PostgreSQL is the database engine we will use.

Adding PostgreSQL to Docker Compose
-----------------------------------

.. index::
    single: Docker;PostgreSQL

On our local machine, we have decided to use Docker to manage services. The generated ``docker-compose.yml`` file already contains PostgreSQL as a service:

.. code-block:: yaml
    :caption: docker-compose.yml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-13}-alpine
        environment:
        POSTGRES_DB: ${POSTGRES_DB:-app}
        # You should definitely change the password in production
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
        POSTGRES_USER: ${POSTGRES_USER:-symfony}
        volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

This will install a PostgreSQL server and configure some environment variables that control the database name and credentials. The values do not really matter.

We also expose the PostgreSQL port (``5432``) of the container to the local host. That will help us access the database from our machine:

.. code-block:: yaml
    :caption: docker-compose.override.yml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    The ``pdo_pgsql`` extension should have been installed when PHP was set up in a previous step.

Starting Docker Compose
-----------------------

Start Docker Compose in the background (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Wait a bit to let the database start up and check that everything is running fine:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

If there are no running containers or if the ``State`` column does not read ``Up``, check the Docker Compose logs:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Accessing the Local Database
----------------------------

Using the ``psql`` command-line utility might prove useful from time to time. But you need to remember the credentials and the database name. Less obvious, you also need to know the local port the database runs on the host. Docker chooses a random port so that you can work on more than one project using PostgreSQL at the same time (the local port is part of the output of ``docker-compose ps``).

If you run ``psql`` via the Symfony CLI, you don't need to remember anything.

The Symfony CLI automatically detects the Docker services running for the project and exposes the environment variables that ``psql`` needs to connect to the database.

.. index::
    single: Symfony CLI;run psql

Thanks to these conventions, accessing the database via ``symfony run`` is much easier:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main main

Dumping and Restoring Database Data
-----------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Use ``pg_dump`` to dump the database data:

.. code-block:: bash
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

And restore the data:

.. code-block:: bash
    :class: ignore

    $ symfony run psql < dump.sql

Adding PostgreSQL to Platform.sh
--------------------------------

.. index::
    single: Platform.sh;PostgreSQL

For the production infrastructure on Platform.sh, adding a service like PostgreSQL should be done in the ``.platform/services.yaml`` file, which was already done via the ``webapp`` package recipe:

.. code-block:: yaml
    :caption: .platform/services.yaml
    :class: ignore

    database:
        type: postgresql:13
        disk: 1024

The ``database`` service is a PostgreSQL database (same version as for Docker) that we want to provision with 1GB of disk.

We also need to "link" the DB to the application container, which is described in ``.platform.app.yaml``:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

The ``database`` service of type ``postgresql`` is referenced as ``database`` on the application container.

Check that the ``pdo_pgsql`` extension is already installed for the PHP runtime:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Accessing the Platform.sh Database
----------------------------------

PostgreSQL is now running both locally via Docker and in production on Platform.sh.

As we have just seen, running ``symfony run psql`` automatically connects to the database hosted by Docker thanks to environment variables exposed by ``symfony run``.

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

If you want to connect to PostgreSQL hosted on the production containers, you can open an SSH tunnel between the local machine and the Platform.sh infrastructure:

.. code-block:: bash
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

By default, Platform.sh services are not exposed as environment variables on the local machine. You must explicitly do so by running the ``var:expose-from-tunnel`` command. Why? Connecting to the production database is a dangerous operation. You can mess with *real* data. Requiring the flag is how you confirm that this *is* what you want to do.

Now, connect to the remote PostgreSQL database via ``symfony run psql`` as before:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

When done, don't forget to close the tunnel:

.. code-block:: bash
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    To run some SQL queries on the production database instead of getting a shell, you can also use the ``symfony sql`` command.

Exposing Environment Variables
------------------------------

.. index::
    single: Platform.sh;Environment Variables
    single: Symfony CLI;var:export

Docker Compose and Platform.sh work seamlessly with Symfony thanks to environment variables.

Check all environment variables exposed by ``symfony`` by executing ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

The ``PG*`` environment variables are read by the ``psql`` utility. What about the others?

When a tunnel is open to Platform.sh with ``var:expose-from-tunnel``, the ``var:export`` command returns remote environment variables:

.. code-block:: bash
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and Platform.sh use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

.. sidebar:: Going Further

    * `Platform.sh services`_;

    * `Platform.sh tunnel`_;

    * `PostgreSQL documentation`_;

    * `docker-compose commands`_.

.. _`Platform.sh services`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Platform.sh tunnel`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL documentation`: https://www.postgresql.org/docs/
.. _`docker-compose commands`: https://docs.docker.com/compose/reference/
