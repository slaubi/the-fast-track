Ramificando el código
======================

Hay muchas maneras de organizar el flujo de trabajo de los cambios en el código de un proyecto. Pero trabajar directamente en la rama maestra de Git y desplegar directamente a la de producción sin realizar pruebas probablemente no sea la mejor.

Hacer pruebas no es sólo hacer pruebas unitarias o pruebas funcionales, sino que también se trata de comprobar el comportamiento de la aplicación con datos de producción. Si tú o el `resto de partes interesadas`_ podéis navegar por la aplicación exactamente como cuando sea desplegada a los usuarios finales, esto se convierte en una gran ventaja y te permite desplegarla con confianza. Es especialmente útil cuando personas sin conocimientos técnicos pueden validar nuevas características.

Continuaremos haciendo todo el trabajo en la rama *master* de Git en los próximos pasos por simplicidad y para evitar repetirnos, pero veamos cómo podría funcionar mejor.

Adoptando un flujo de trabajo en Git
------------------------------------

Un posible flujo de trabajo es crear una rama por cada nueva característica o corrección de errores. Es simple y eficiente.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Creando ramas
-------------

.. index::
    single: Git;branch
    single: Git;checkout

El flujo de trabajo comienza con la creación de una rama de Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Este comando crea la rama ``sessions-in-redis``  desde la rama ``master``. Se "bifurca" el código y la configuración de la infraestructura.

Almacenando sesiones en Redis
-----------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Como habrás adivinado por el nombre de la rama, queremos cambiar el almacenamiento de sesión del sistema de ficheros a un almacén Redis.

Los pasos necesarios para hacerlo realidad son típicos:

#. Crea una rama de Git;

#. Actualiza la configuración de Symfony si es necesario;

#. Escribe y/o actualiza alguna parte del código si es necesario;

#. Actualiza la configuración de PHP (agrega la extensión Redis PHP);

#. Actualiza la infraestructura en Docker y SymfonyCloud (agrega el servicio Redis);

#. Prueba localmente;

#. Prueba remotamente;

#. Fusiona la rama con *master*;

#. Despliega a producción;

#. Elimina la rama.

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

Te dejaré hacer pruebas localmente navegando por el sitio web. Como no hay cambios visuales y como aún no estamos utilizando las sesiones, todo debería funcionar como antes.

Desplegando una rama
--------------------

.. index::
    single: SymfonyCloud;Environment

Antes de desplegar en producción, debemos probar la rama en la misma infraestructura que tiene producción. También debemos validar que todo funciona bien para el entorno de Symfony ``prod`` (el sitio web local utiliza el entorno de Symfony ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Ahora, vamos a crear un *entorno SymfonyCloud* basado en la *rama de Git*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Este comando crea un nuevo entorno de la siguiente manera:

* La rama hereda el código y la infraestructura de la rama actual de Git (``sessions-in-redis``);

* Los datos vienen del entorno *master* (también conocido como producción) tomando una instantánea consistente de todos los datos del servicio, incluidos los archivos (por ejemplo, los archivos cargados por el usuario) y las bases de datos;

* Se crea un nuevo clúster dedicado para desplegar el código, los datos y la infraestructura.

Como la implementación sigue los mismos pasos que el despliegue en producción, también se ejecutarán las migraciones de bases de datos. Esta es una excelente manera de validar que las migraciones funcionen con los datos de producción.

Los entornos que no son ``master`` son muy similares al ``master`` anterior, excepto por algunas pequeñas diferencias: por ejemplo, los correos electrónicos no se envían por defecto.

.. index::
    single: Symfony CLI;open:remote

Una vez finalizado el despliegue, abre la nueva rama en un navegador:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Ten en cuenta que todos los comandos de SymfonyCloud funcionan en la rama actual de Git. Este comando abre la URL que ha sido desplegada para la rama ``sessions-in-redis``; la URL será algo similar a ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Prueba el sitio web en este nuevo entorno, deberías ver todos los datos que creaste en el entorno *master*.

Si se añaden más conferencias en el entorno ``master``, no aparecerán en el entorno ``sessions-in-redis`` y viceversa. Los entornos son independientes y aislados.

Si el código se desarrolla en *master*, siempre puedes hacer *rebase* a la rama de Git y desplegar la versión actualizada, resolviendo los conflictos tanto para el código como para la infraestructura.

.. index::
    single: Symfony CLI;env:sync

Incluso puedes sincronizar los datos desde la rama *master* con el entorno ``sessions-in-redis``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Depurando las implementaciones de producción antes del despliegue
------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Por defecto, todos los entornos SymfonyCloud utilizan la misma configuración que el entorno ``master``/``prod`` (también conocido como entorno Symfony ``prod``). Esto te permite probar la aplicación en condiciones reales. Te da la sensación de desarrollar y probar directamente en servidores de producción, pero sin los riesgos asociados con ello. Esto me recuerda a los buenos tiempos en los que estábamos realizando la implementación a través de FTP.

.. index::
    single: Symfony CLI;env:debug

En caso que tengas algún problema, es posible que quieras cambiar al entorno Symfony ``dev``:

.. code-block:: bash

    $ symfony env:debug

Cuando hayas terminado, vuelve a los ajustes de producción:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Nunca** habilites el entorno ``dev`` y nunca habilites el *Symfony Profiler* en la rama ``master``; esto haría tu aplicación realmente lenta y abriría muchas vulnerabilidades serias de seguridad.

Probando las implementaciones de producción antes del despliegue
-----------------------------------------------------------------

Tener acceso a la próxima versión del sitio web con datos de producción te abre un abanico de oportunidades: desde pruebas visuales de regresión hasta pruebas de rendimiento. `Blackfire <https://blackfire.io>`_ es la herramienta perfecta para este trabajo.

Consulta el paso "Rendimiento" para obtener más información sobre cómo puedes utilizar Blackfire para probar tu código antes del despliegue.

Fusionando en producción
-------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Cuando estés satisfecho con los cambios en la rama, vuelve a fusionar el código y la infraestructura con la rama master de Git:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

Y haz el despliegue:

.. code-block:: bash

    $ symfony deploy

Cuando se despliega, sólo los cambios de código e infraestructura son enviados a SymfonyCloud; los datos no se ven afectados de ninguna manera.

Limpiando
---------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Finalmente, elimina la rama Git y el entorno SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Yendo más allá

    * `Ramificación en Git; <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_

    * `Redis docs <https://redis.io/documentation>`_.

.. _`resto de partes interesadas`: https://en.wikipedia.org/wiki/Project_stakeholder
