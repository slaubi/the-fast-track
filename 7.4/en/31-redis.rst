Using Redis to Store Sessions
=============================

.. index::
    single: Redis

Depending on the website traffic and/or its infrastructure, you might want to use Redis to manage user sessions instead of PostgreSQL.

When we talked about branching the project's code to move sessions from the filesystem to the database, we listed all the needed step to add a new service.

Here is how you can add Redis to your project in one patch:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Upsun;Redis

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -37,6 +37,7 @@ applications:
                     - iconv
                     - mbstring
                     - pdo_pgsql
    +                - redis
                     - sodium
                     - xsl

    @@ -62,6 +63,7 @@ applications:

             relationships:
                 database: "database:postgresql"
    +            redis: "rediscache:redis"

             hooks:
                 build: |
    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -22,6 +22,9 @@ services:
                 type: network-storage:2.0
                 disk: 256

    +    rediscache:
    +        type: redis:8.0
    +
     applications:
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:8.0-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -4,3 +4,3 @@ framework:
         # Note that the session will be started ONLY if you read or write from it.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

Test locally by browsing the website; everything should still work as before.

Commit and deploy as usual:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Going Further

    * `Redis docs`_.

.. _`Redis docs`: https://redis.io/documentation
