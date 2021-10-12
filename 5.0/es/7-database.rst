Creando una base de datos
=========================

.. index::
    single: Database

El sitio web del Libro de Visitas de la Conferencia trata sobre la recopilación de comentarios durante las conferencias. Necesitamos almacenar los comentarios aportados por los asistentes a la conferencia en un almacenamiento permanente.

Un comentario se describe mejor con una estructura de datos fija: un autor, su correo electrónico, el texto del comentario y una foto opcional. Es el tipo de datos que mejor se puede almacenar en un motor de base de datos relacional tradicional.

PostgreSQL es el motor de base de datos que usaremos.

Añadiendo PostgreSQL a Docker Compose
--------------------------------------

.. index::
    single: Docker;PostgreSQL

En nuestra máquina local, hemos decidido utilizar Docker para gestionar los servicios. Crea un fichero ``docker-compose.yaml`` y añade PostgreSQL como un servicio:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

Esto instalará un servidor PostgreSQL en su versión 11 y configurará algunas variables de entorno para controlar el nombre y las credenciales de la base de datos. Los valores no importan realmente.

También abriremos el puerto PostgreSQL (``5432``) del contenedor a la máquina local. Eso nos ayudará a acceder a la base de datos desde nuestra máquina.

.. note::

    La extensión ``pdo_pgsql`` debería haber sido instalada cuando se configuró PHP en un paso anterior.

Iniciando Docker Compose
------------------------

Iniciar Docker Compose en segundo plano (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Espera un poco para que la base de datos se inicie y comprueba que todo funciona correctamente:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Si no hay contenedores en marcha o si en la columna ``State`` no dice ``Up``, comprueba los *logs* de Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Accediendo a la base de datos local
-----------------------------------

Usar la utilidad de línea de comandos de ``psql`` puede resultar útil de vez en cuando, pero necesitas recordar las credenciales y el nombre de la base de datos. Menos obvio aún es que también necesitas saber el puerto local en el que se ejecuta la base de datos en el host. Docker elige un puerto aleatorio para que puedas trabajar en más de un proyecto usando PostgreSQL al mismo tiempo (el puerto local es parte de la salida de ``docker-compose ps``).

Si se ejecuta ``psql`` a través de la interfaz de línea de comandos de Symfony, no necesitas recordar nada.

La interfaz de línea de comandos de Symfony detecta automáticamente los servicios Docker en ejecución para el proyecto y expone las variables de entorno que ``psql`` necesita para conectarse a la base de datos.

.. index::
    single: Symfony CLI;run psql

Gracias a estas convenciones, es mucho más fácil acceder a la base de datos a través de ``symfony run``:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Añadiendo PostgreSQL a SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Para la infraestructura de producción en SymfonyCloud, añadir un servicio como PostgreSQL debería hacerse en el archivo ``.symfony/services.yaml``, que ahora mismo está vacío:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

El servicio ``db`` es una base de datos PostgreSQL 11 (como para Docker) que queremos aprovisionar en un pequeño contenedor con 1GB de disco.

También necesitamos "enlazar" la DB con el contenedor de la aplicación, que se describe en ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

El servicio ``db`` de tipo ``postgresql`` es referenciado como ``database`` en el contenedor de aplicación.

El último paso es añadir la extensión ``pdo_pgsql`` al entorno de ejecución de PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Aquí está el *diff* completo de los cambios hechos en ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

Haz commit de estos cambios y vuelve a desplegar a SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Accediendo a la base de datos SymfonyCloud
------------------------------------------

PostgreSQL se está ejecutando ahora tanto localmente a través de Docker como en producción en SymfonyCloud.

Como acabamos de ver, al ejecutarse ``symfony run psql`` se conecta automáticamente a la base de datos alojada por Docker gracias a las variables de entorno expuestas por ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Si deseas conectarte al PostgreSQL que está alojado en los contenedores de producción, puedes abrir un túnel SSH entre la máquina local y la infraestructura SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

Por defecto, los servicios SymfonyCloud no están expuestos como variables de entorno en el equipo local. Debes hacerlo explícitamente utilizando la opción ``--expose-env-vars``. ¿Por qué? Conectarse a la base de datos de producción es una operación peligrosa. Puedes lidiar con datos *reales*. Requerir esa opción es la forma de confirmar que eso *es* lo que quieres hacer.

Ahora, conéctate a la base de datos remota de PostgreSQL con ``symfony run psql`` como antes:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Cuando termines, no olvides cerrar el túnel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Para ejecutar algunas consultas SQL en la base de datos de producción en lugar de obtener una shell, también puedes utilizar el comando ``symfony sql``.

Exponiendo variables de entorno
-------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose y SymfonyCloud funcionan perfectamente con Symfony gracias a las variables de entorno.

Verifica todas las variables de entorno expuestas por ``symfony`` ejecutando ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Las variables de entorno ``PG*`` se leen por la utilidad ``psql``. ¿Qué hay de las otras?

Cuando tienes abierto un túnel a SymfonyCloud con la opción ``--expose-env-vars`` activada, el comando ``var:export`` devuelve las variables de entorno remotas:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: Yendo más allá

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Documentación de PostgreSQL <https://www.postgresql.org/docs/>`_ ;

    * `Comandos de docker-compose <https://docs.docker.com/compose/reference/>`_.
