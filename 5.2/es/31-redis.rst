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
    single: SymfonyCloud;Redis

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - redis
             - blackfire
             - xsl
             - pdo_pgsql
    @@ -26,6 +27,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
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

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -17,3 +17,7 @@ services:
             image: blackfire/blackfire
             env_file: .env.local
             ports: [8707]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

¿No es *hermoso*?

"Reinicia" Docker para iniciar el servicio Redis:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Prueba localmente navegando por el sitio web; todo debería seguir funcionando como antes.

Confirma y despliegua como de costumbre:

.. code-block:: bash
    :class: ignore

    $ symfony deploy

.. sidebar:: Yendo más allá

    * `Documentación Redis <https://redis.io/documentation>`_.
