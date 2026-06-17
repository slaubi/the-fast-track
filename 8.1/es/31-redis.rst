Utilizando Redis para almacenar sesiones
========================================

.. index::
    single: Redis

Según el tráfico del sitio web y/o su infraestructura, es posible que desees utilizar Redis para administrar las sesiones de usuario en lugar de PostgreSQL.

Cuando hablamos de bifurcar el código del proyecto para mover la sesión del sistema de archivos a la base de datos, enumeramos todos los pasos necesarios para agregar un nuevo servicio.

Así es como puedes agregar Redis a tu proyecto en un parche:

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
    @@ -21,3 +21,6 @@ services:
                 type: network-storage:2.0

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

¿No es *hermoso*?

"Reinicia" Docker para iniciar el servicio Redis:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Prueba localmente navegando por el sitio web; todo debería seguir funcionando como antes.

Confirma y despliegua como de costumbre:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: Yendo más allá

    * `Documentación Redis <https://redis.io/documentation>`_.
