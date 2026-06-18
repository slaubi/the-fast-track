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

En nuestra máquina local, hemos decidido utilizar Docker para gestionar los servicios. El fichero ``compose.yaml`` generado ya contiene PostgreSQL como un servicio:

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

Esto instalará un servidor PostgreSQL y configurará algunas variables de entorno para controlar el nombre y las credenciales de la base de datos. Los valores no importan realmente.

También abriremos el puerto PostgreSQL (``5432``) del contenedor a la máquina local. Eso nos ayudará a acceder a la base de datos desde nuestra máquina:

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

    La extensión ``pdo_pgsql`` debería haber sido instalada cuando se configuró PHP en un paso anterior.

Iniciando Docker Compose
------------------------

Iniciar Docker Compose en segundo plano (``-d``):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

Espera un poco para que la base de datos se inicie y comprueba que todo funciona correctamente:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Si no hay contenedores en marcha o si en la columna ``State`` no dice ``Up``, comprueba los *logs* de Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

Accediendo a la base de datos local
-----------------------------------

Usar la utilidad de línea de comandos de ``psql`` puede resultar útil de vez en cuando, pero necesitas recordar las credenciales y el nombre de la base de datos. Menos obvio aún es que también necesitas saber el puerto local en el que se ejecuta la base de datos en el host. Docker elige un puerto aleatorio para que puedas trabajar en más de un proyecto usando PostgreSQL al mismo tiempo (el puerto local es parte de la salida de ``docker-compose ps``).

Si se ejecuta ``psql`` a través de la interfaz de línea de comandos de Symfony, no necesitas recordar nada.

La interfaz de línea de comandos de Symfony detecta automáticamente los servicios Docker en ejecución para el proyecto y expone las variables de entorno que ``psql`` necesita para conectarse a la base de datos.

.. index::
    single: Symfony CLI;run psql

Gracias a estas convenciones, es mucho más fácil acceder a la base de datos a través de ``symfony run``:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Si no tienes el binario de ``psql`` en tu máquina local, también puedes ejecutarlo a través de ``docker compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

Volcado y restauración de la base de datos
-------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Utiliza ``pg_dump`` para volcar los datos de la base de datos:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

Y restaura los datos:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Añadiendo PostgreSQL a Upsun
------------------------------------

.. index::
    single: Upsun;PostgreSQL

Para la infraestructura de producción en Upsun, añadir un servicio como PostgreSQL debería hacerse en el archivo ``.upsun/config.yaml``, lo cual ya se hizo a través de la receta del paquete ``webapp``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

El servicio ``database`` es una base de datos PostgreSQL (la misma versión que para Docker). Upsun asigna su disco automáticamente en el primer despliegue; ajústalo más tarde con ``symfony cloud:resources:set`` si es necesario.

También necesitamos "enlazar" la DB con el contenedor de la aplicación, que se describe en ``.upsun/config.yaml``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

El servicio ``database`` de tipo ``postgresql`` es referenciado como ``database`` en el contenedor de aplicación.

Comprueba que la extensión ``pdo_pgsql`` ya está instalada para el entorno de ejecución de PHP:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Accediendo a la base de datos Upsun
------------------------------------------

PostgreSQL se está ejecutando ahora tanto localmente a través de Docker como en producción en Upsun.

Como acabamos de ver, al ejecutarse ``symfony run psql`` se conecta automáticamente a la base de datos alojada por Docker gracias a las variables de entorno expuestas por ``symfony run``.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Si deseas conectarte al PostgreSQL que está alojado en los contenedores de producción, puedes abrir un túnel SSH entre la máquina local y la infraestructura Upsun:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Por defecto, los servicios Upsun no están expuestos como variables de entorno en el equipo local. Debes hacerlo explícitamente ejecutando el comando ``var:expose-from-tunnel``. ¿Por qué? Conectarse a la base de datos de producción es una operación peligrosa. Puedes lidiar con datos *reales*.

Ahora, conéctate a la base de datos remota de PostgreSQL con ``symfony run psql`` como antes:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Cuando termines, no olvides cerrar el túnel:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Para ejecutar algunas consultas SQL en la base de datos de producción en lugar de obtener una shell, también puedes utilizar el comando ``symfony sql``.

Exponiendo variables de entorno
-------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

Docker Compose y Upsun funcionan perfectamente con Symfony gracias a las variables de entorno.

Verifica todas las variables de entorno expuestas por ``symfony`` ejecutando ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Las variables de entorno ``PG*`` se leen por la utilidad ``psql``. ¿Qué hay de las otras?

Cuando tienes abierto un túnel a Upsun con ``var:expose-from-tunnel``, el comando ``var:export`` devuelve las variables de entorno remotas:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Describiendo tu Infraestructura
-------------------------------

Es posible que todavía no te hayas dado cuenta, pero tener la infraestructura almacenada en ficheros junto al código es muy útil. Docker y Upsun utilizan archivos de configuración para describir la infraestructura del proyecto. Cuando necesitamos un nuevo servicio adicional, los cambios en el código y los cambios en la infraestructura son parte del mismo parche.

.. sidebar:: Yendo más allá

    * `Servicios de Upsun`_ ;

    * `Túnel Upsun`_ ;

    * `Documentación de PostgreSQL`_ ;

    * los `Comandos de Docker Compose`_ .

.. _`Servicios de Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Túnel Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Documentación de PostgreSQL`: https://www.postgresql.org/docs/
.. _`Comandos de Docker Compose`: https://docs.docker.com/reference/cli/docker/compose/
