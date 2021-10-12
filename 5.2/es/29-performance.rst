Gestionando el rendimiento
==========================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Optimizar antes de tiempo es la raíz de todos los males.

Tal vez ya hayas leído esta cita antes. Pero me gusta citarla en su totalidad:

.. epigraph::

    Deberíamos olvidar las pequeñas optimizaciones el 97 % de las veces; optimizar antes de tiempo es el origen de todos los males. Sin embargo, no deberíamos dejar pasar la oportunidad de optimizar ese crucial 3 %.

    --   Donald Knuth

Incluso pequeñas mejoras en el rendimiento pueden marcar la diferencia, especialmente para sitios de comercio electrónico. Ahora que la aplicación del libro de visitas está lista para el gran público, veamos cómo podemos comprobar su rendimiento.

La mejor manera de encontrar optimizaciones de rendimiento es usar un analizador de rendimiento o *perfilador* (*profiler*). La opción más popular hoy en día es `Blackfire <https://blackfire.io>`_ (*aviso a navegantes*: también soy el fundador del proyecto Blackfire).

Introducción a Blackfire
------------------------

Blackfire se compone de varias partes:

* Un *cliente* que activa perfiles (la herramienta CLI de Blackfire o una extensión de navegador para Google Chrome o Firefox);

* Un *agente* que prepara y agrega datos antes de enviarlos a blackfire.io para su visualización;

* Una extensión PHP (que actúa a modo de *sonda*) y que analiza el código PHP.

Para trabajar con Blackfire, primero tienes que `registrarte <https://blackfire.io/signup>`_ .

Instala Blackfire en tu equipo local, ejecutando el siguiente script de instalación rápida:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Este instalador descarga la herramienta de línea de comandos de Blackfire, y luego instala la sonda PHP (sin habilitarla) en todas las versiones disponibles de PHP.

Habilita la sonda PHP para nuestro proyecto:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Reinicia el servidor web para que PHP pueda cargar Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

La herramienta de línea de comandos de Blackfire necesita estar configurada con tus credenciales personales de **cliente** (para almacenar los perfiles de tus proyectos en tu cuenta personal). Encuéntralas en la parte superior de la `página ``Settings/Credentials`` (configuración/credenciales) <https://blackfire.io/my/settings/credentials>`_ y ejecuta el siguiente comando reemplazando los marcadores de posición:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Para obtener instrucciones completas de instalación, sigue la `guía oficial de instalación detallada <https://blackfire.io/docs/up-and-running/installation>`_. Son muy útiles cuando se instala Blackfire en un servidor.

Configuración del agente Blackfire en Docker
--------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

El último paso es añadir el servicio del agente Blackfire en el *stack* (pila) de servicios de Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Para comunicarse con el servidor, necesitas obtener tus credenciales personales del **servidor** (estas credenciales identifican dónde deseas almacenar los perfiles; puedes crear una por proyecto); se pueden encontrar en la parte inferior de la página ``Settings/Credentials``  `<https://blackfire.io/my/settings/credentials>`_. Almacénalas en un archivo local ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Ahora puedes lanzar el nuevo contenedor:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Arreglando una instalación de Blackfire que no funciona
-------------------------------------------------------

Si obtienes un error durante el análisis de rendimiento, aumenta el nivel de registro de Blackfire para obtener más información:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Reinicia el servidor web:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Y revisa los *logs* en vivo (en tiempo real):

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Analiza la aplicación de nuevo y comprueba la salida del *log*.


Configurando Blackfire en producción
------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire está incluido por defecto en todos los proyectos de SymfonyCloud.

Configura las credenciales del *servidor* como variables de entorno:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Y habilita la sonda PHP como cualquier otra extensión PHP:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - pdo_pgsql
             - apcu

Configurando Varnish para Blackfire
-----------------------------------

.. index::
    single: SymfonyCloud;Varnish

Antes de que puedas desplegar para empezar a realizar las tareas de análisis, necesitas una forma de evitar la caché HTTP de Varnish. Si no, Blackfire nunca llegará a la aplicación PHP. Vas a autorizar solo las peticiones de *profiling* que provengan de tu máquina local.

Encuentra tu dirección IP actual:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

y úsala para configurar Varnish:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

Ahora puedes realizar el despliegue.

Analizando el rendimiento de páginas web
----------------------------------------

.. index::
    single: Profiling;Web Pages

Puedes hacer análisis de páginas web tradicionales desde Firefox o Google Chrome a través de sus `extensiones dedicadas <https://blackfire.io/docs/integrations/browsers/index>`_.

No olvides desactivar la caché HTTP en ``config/packages/framework.yaml`` cuando realices el análisis en tu máquina local: si no lo haces, analizarás la capa de caché HTTP de Symfony en lugar de tu propio código:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -16,4 +16,4 @@ framework:
         php_errors:
             log: true

    -    http_cache: true
    +    #http_cache: true

Para tener una mejor idea del rendimiento de tu aplicación en producción, deberías también hacer un análisis del entorno de "producción". Por defecto, tu entorno local utiliza el entorno de "desarrollo", que añade una sobrecarga significativa (principalmente para recopilar datos para la barra de herramientas de depuración web y el analizador de rendimiento de Symfony).

.. index::
    single: Symfony CLI;server:prod

El cambio de tu máquina local al entorno de producción se puede hacer cambiando la variable de entorno ``APP_ENV`` en el archivo ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

O puedes usar el comando ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

No olvides volver a cambiar al entorno de desarrollo cuando finalice tu sesión de *profiling* (análisis de rendimiento):

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Analizando el rendimiento de los recursos de la API
---------------------------------------------------

.. index::
    single: Profiling;API

El análisis de rendimiento de la API o la SPA se realiza mejor en la línea de comandos usando la herramienta Blackfire que se ha instalado previamente:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

El comando ``blackfire curl`` acepta exactamente los mismos argumentos y opciones que `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Comparación del rendimiento
---------------------------

En el paso acerca de la "Caché" añadimos una capa de caché para mejorar el rendimiento de nuestro código, pero no hemos comprobado ni medido el impacto en el rendimiento que ha tenido el cambio. Como las personas somos poco efectivas adivinando qué será rápido y qué será lento, podrías terminar en una situación en la que añadir un poco de optimización hiciera que tu aplicación fuera, realmente, más lenta.

Deberías medir siempre el impacto de cualquier optimización que hagas con un analizador de rendimiento. Blackfire lo hace visualmente más fácil gracias a su `función de comparación <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Escribiendo pruebas funcionales de caja negra
---------------------------------------------

.. index::
    single: Blackfire;Player

Hemos visto cómo escribir pruebas funcionales con Symfony. Blackfire se puede utilizar para escribir escenarios de navegación que se pueden ejecutar bajo demanda a través del `Blackfire player <https://blackfire.io/player>`_. Escribamos un escenario que envíe un nuevo comentario y lo valide en desarrollo a través del enlace de correo electrónico y a través del admin en producción.

Crea un archivo de nombre ``.blackfire.yaml`` con el siguiente contenido:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Descarga el reproductor Blackfire para poder ejecutar el escenario localmente:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Ejecuta este escenario en desarrollo:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

O en producción:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Los escenarios de Blackfire también pueden desencadenar perfiles para cada solicitud y ejecutar pruebas de rendimiento añadiendo la opción ``--blackfire``.

Automatizando los análisis de rendimiento
-----------------------------------------

La gestión del rendimiento no sólo consiste en mejorar el rendimiento del código existente, sino también en comprobar que no se introducen regresiones de rendimiento.

El escenario escrito en la sección anterior puede ejecutarse automáticamente en un *workflow* de integración continua (CI), o en producción, de forma regular.

En SymfonyCloud, también se permite `ejecutar los escenarios <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ cada vez que se crea una nueva rama o se despliega en producción para analizar automáticamente el rendimiento del nuevo código.

.. sidebar:: Yendo más allá

    * `El libro de Blackfire: Rendimiento del código PHP a fondo
<https://blackfire.io/book>`_;

    * `Tutorial de Blackfire en SymfonyCasts <https://symfonycasts.com/screencast/blackfire>`_.
