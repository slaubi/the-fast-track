Revisando tu entorno de trabajo
===============================

Antes de empezar a trabajar en el proyecto, tenemos que comprobar que todo el mundo tiene un buen entorno de trabajo. Es muy importante. Las herramientas de desarrollo que tenemos a nuestra disposición hoy en día son muy diferentes de las que teníamos hace 10 años. Han evolucionado mucho, para mejor. Sería una pena no aprovecharlas. Las buenas herramientas te pueden llevar lejos en poco tiempo.

Por favor, no te saltes este paso. O al menos, lee la última sección sobre la CLI de Symfony.

Un ordenador
------------

Necesitas un ordenador. La buena noticia es que Symfony puede funcionar en cualquiera de los sistemas operativos más conocidos: macOS, Windows o Linux. Symfony y todas las herramientas que vamos a utilizar son compatibles con cada uno de ellos.

Decisiones subjetivas
---------------------

Quiero ir rápido con las mejores opciones. He tomado decisiones subjetivas para este libro.

`PostgreSQL`_ va a ser nuestra elección para todo: desde la base de datos hasta las colas, desde la caché hasta el almacenamiento en sesión. Para la mayoría de los proyectos, PostgreSQL es la mejor solución, tiene gran escalabilidad y permite simplificar la infraestructura administrando solo un único servicio.

Al final del libro, aprenderemos a gestionar las colas de mensajes con `RabbitMQ`_ y las sesiones con `Redis`_ .

IDE
---

.. index:: IDE

Puedes utilizar el Bloc de notas si quieres. Aunque no te lo recomiendo.

Solía trabajar con Textmate. Ya no. La comodidad de utilizar un IDE "real" no tiene precio. Auto-completado, agregar y ordenar automáticamente las sentencias ``use`` o saltar de un archivo a otro son algunas de las características que aumentarán tu productividad.

Recomendaría usar `Visual Studio Code`_ o `PhpStorm`_ . El primero es gratuito, el segundo no lo es pero tiene una mejor integración con Symfony (gracias al `plugin de soporte de Symfony`_ ). Depende de ti. Sé que quieres saber qué IDE estoy usando. Estoy escribiendo este libro en Visual Studio Code.

Terminal
--------

.. index:: Terminal

Cambiaremos del IDE a la línea de comandos todo el tiempo. Puedes usar el terminal incorporado de tu IDE, pero yo prefiero usar uno real para tener más espacio.

Linux viene con un ``Terminal`` incorporado. Utiliza `iTerm2`_ en macOS. En Windows, `Hyper`_ funciona bien.

Git
---

.. index:: Git

Mi último libro recomendaba Subversion para el control de versiones. Parece que todo el mundo está usando `Git`_ ahora.

En Windows, instala `Git bash`_ .

Asegúrate de que sabes cómo realizar las operaciones más comunes, como ejecutar ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``...

PHP
---

.. index:: PHP

Usaremos Docker para los servicios, pero me gusta tener PHP instalado en mi ordenador local por razones de rendimiento, estabilidad y simplicidad. Puedes pensar que soy de la vieja escuela, pero la combinación de un servicio local de PHP y Docker es la combinación perfecta para mí.

Usa PHP 8.0 y comprueba que las :index:`siguientes extensiones de PHP <PHP extensions>` están instaladas o instálalas ahora: ``intl`` , ``pdo_pgsql`` , ``xsl`` , ``amqp`` , ``gd`` , ``openssl`` y ``sodium``. Opcionalmente también puedes instalar ``redis``, ``curl`` y ``zip``.

Puedes comprobar las extensiones habilitadas actualmente usando ``php -m``.

También necesitamos ``php-fpm`` si tu plataforma lo soporta, ``php-cgi`` también sirve.

Composer
--------

.. index:: Composer

Hoy en día, gestionar dependencias lo es todo en un proyecto Symfony. Descarga la última versión de `Composer`_ , la herramienta de gestión de paquetes para PHP.

Si no estás familiarizado con Composer, tómate un tiempo para leer sobre él.

.. tip::

    No es necesario que escribas los nombres completos de los comandos: ``composer req`` hace lo mismo que ``composer require``, usa ``composer rem`` en lugar de ``composer remove``...

NodeJS
------

No escribiremos mucho código JavaScript, pero usaremos herramientas JavaScript / NodeJS para administrar nuestros assets. Comprueba que tienes `NodeJS`_ instalado y el gestor de paquetes `Yarn`_.

Docker y Docker Compose
-----------------------

.. index:: Docker,Docker Compose

Gestionaremos los servicios mediante Docker y Docker Compose. `Instálalos`_ e inicia Docker. Si es la primera vez que lo utilizas, familiarízate con él. Pero no te asustes, lo usaremos de forma muy sencilla. Sin configuraciones sofisticadas ni complejas.

Symfony CLI
-----------

.. index:: Symfony CLI

Por último, pero no por ello menos importante, utilizaremos la línea de comandos (CLI) de ``symfony`` para aumentar nuestra productividad. Desde el servidor web local que proporciona, hasta la integración completa con Docker y el soporte de Upsun, nos servirá para ahorrar mucho tiempo.

Instala `Symfony CLI`_ y muévelo a algún lugar dentro de tu ``$PATH``. Crea una cuenta `SymfonyConnect`_ si aún no la tienes e inicia sesión a través de ``symfony login``.

Para utilizar HTTPS localmente, también necesitamos `instalar una autoridad de certificación (CA)`_ para habilitar el soporte de TLS. Ejecuta el siguiente comando:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: terminal
    :class: ignore

    $ symfony server:ca:install

Comprueba que tu ordenador tiene todos los requisitos necesarios ejecutando el siguiente comando:

.. code-block:: terminal
    :class: ignore

    $ symfony book:check-requirements

Si quieres trabajar de forma aún más elegante, también puedes ejecutar el `proxy de Symfony`_ . Es opcional, pero te permite obtener un nombre de dominio local que termina en ``.wip`` para tu proyecto.

Cuando ejecutemos un comando en una terminal, casi siempre le pondremos el prefijo ``symfony``, por ejemplo ``symfony composer``, en vez de sólo ``composer`` , o ``symfony console`` en lugar de ``./bin/console``.

La razón principal es que la CLI de Symfony establece automáticamente algunas variables de entorno basadas en los servicios que se ejecutan en tu máquina a través de Docker. Estas variables de entorno están disponibles para las peticiones HTTP porque el servidor web local las inyecta automáticamente. Por lo tanto, el uso de ``symfony`` en la CLI asegura que tengas el mismo comportamiento en todos los componentes de tu entorno.

Además, la CLI de Symfony selecciona automáticamente la "mejor" versión posible de PHP para el proyecto.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`NodeJS`: https://nodejs.org/
.. _`Yarn`: https://classic.yarnpkg.com/en/docs/install/
.. _`Instálalos`: https://docs.docker.com/install/
.. _`Symfony CLI`: https://symfony.com/download
.. _`SymfonyConnect`: https://symfony.com/connect/login
.. _`instalar una autoridad de certificación (CA)`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`plugin de soporte de Symfony`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`proxy de Symfony`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
