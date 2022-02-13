Using Redis to Store Sessions
=============================

.. index::
    single: Redis

Depending on the website traffic and/or its infrastructure, you might want to use Redis to manage user sessions instead of PostgreSQL.

When we talked about branching the project's code to move session from the filesystem to the database, we listed all the needed step to add a new service.

Here is how you can add Redis to your project in one patch:

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Platform.sh;Redis

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -14,6 +14,7 @@ runtime:
             - iconv
             - mbstring
             - pdo_pgsql
    +        - redis
             - sodium
             - xsl

    @@ -39,6 +40,7 @@ mounts:

     relationships:
         database: "database:postgresql"
    +    redis: "rediscache:redis"

     hooks:
         build: |
    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -15,3 +15,6 @@ varnish:
     files:
         type: network-storage:1.0
         disk: 256
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: '%env(DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax
             storage_factory_id: session.storage.factory.native
    --- a/docker-compose.yml
    +++ b/docker-compose.yml
    @@ -15,6 +15,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:5-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       db-data:

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

Test locally by browsing the website; everything should still work as before.

Commit and deploy as usual:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy

.. sidebar:: Going Further

    * `Redis docs`_.

.. _`Redis docs`: https://redis.io/documentation
