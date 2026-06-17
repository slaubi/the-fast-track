Ramificando el código
======================

Hay muchas maneras de organizar el flujo de trabajo de los cambios en el código de un proyecto. Pero trabajar directamente en la rama maestra de Git y desplegar directamente a la de producción sin realizar pruebas probablemente no sea la mejor.

Hacer pruebas no es sólo hacer pruebas unitarias o pruebas funcionales, sino que también se trata de comprobar el comportamiento de la aplicación con datos de producción. Si tú o el `resto de partes interesadas`_ podéis navegar por la aplicación exactamente como cuando sea desplegada a los usuarios finales, esto se convierte en una gran ventaja y te permite desplegarla con confianza. Es especialmente útil cuando personas sin conocimientos técnicos pueden validar nuevas características.

Continuaremos haciendo todo el trabajo en la rama *master* de Git en los próximos pasos por simplicidad y para evitar repetirnos, pero veamos cómo podría funcionar mejor.

Adoptando un flujo de trabajo en Git
------------------------------------

Un posible flujo de trabajo es crear una rama por cada nueva característica o corrección de errores. Es simple y eficiente.

Creando ramas
-------------

.. index::
    single: Git;branch
    single: Git;checkout

El flujo de trabajo comienza con la creación de una rama de Git:

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

Este comando crea la rama ``sessions-in-db``  desde la rama ``master``. Se "bifurca" el código y la configuración de la infraestructura.

Almacenando sesiones en la base de datos
----------------------------------------

.. index::
    single: Session;Database

Como habrás adivinado por el nombre de la rama, queremos cambiar el almacenamiento de sesión del sistema de ficheros a un almacén de la base de datos (aquí nuestra base de datos PostgreSQL).

Los pasos necesarios para hacerlo realidad son típicos:

#. Crea una rama de Git;

#. Actualiza la configuración de Symfony si es necesario;

#. Escribe y/o actualiza alguna parte del código si es necesario;

#. Actualiza la configuración de PHP (agrega la extensión PostgreSQL PHP);

#. Actualiza la infraestructura en Docker y Upsun si es necesario (agrega el servicio PostgreSQL);

#. Prueba localmente;

#. Prueba remotamente;

#. Fusiona la rama con *master*;

#. Despliega a producción;

#. Elimina la rama.

Para almacenar sesiones en la base de datos, cambia la configuración ``session.handler_id`` para que apunte al DSN de la base de datos:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -3,7 +3,8 @@ framework:
         secret: '%env(APP_SECRET)%'

         # Note that the session will be started ONLY if you read or write from it.
    -    session: true
    +    session:
    +        handler_id: '%env(resolve:DATABASE_URL)%'

         #esi: true
         #fragments: true

Para almacenar sesiones en la base de datos, es necesario crear la tabla ``sessions``. Hazlo con una migración de Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Migra la base de datos:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Haz pruebas localmente navegando por el sitio web. Como no hay cambios visuales y como aún no estamos utilizando las sesiones, todo debería funcionar como antes.

.. note::

    No necesitamos los pasos 3 a 5 aquí ya que estamos reutilizando la base de datos como almacenamiento de sesión, pero el capítulo sobre el uso de Redis muestra lo sencillo que es agregar, probar e implementar un nuevo servicio tanto en Docker como en Upsun.

Haz commit de tus cambios a la nueva rama:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Desplegando una rama
--------------------

.. index::
    single: Upsun;Environment

Antes de desplegar en producción, debemos probar la rama en la misma infraestructura que tiene producción. También debemos validar que todo funciona bien para el entorno de Symfony ``prod`` (el sitio web local utiliza el entorno de Symfony ``dev``).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Ahora, vamos a crear un *entorno Upsun* basado en la *rama de Git*:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:push

Este comando crea un nuevo entorno de la siguiente manera:

* La rama hereda el código y la infraestructura de la rama actual de Git (``sessions-in-db``);

* Los datos vienen del entorno *master* (también conocido como producción) tomando una instantánea consistente de todos los datos del servicio, incluidos los archivos (por ejemplo, los archivos cargados por el usuario) y las bases de datos;

* Se crea un nuevo clúster dedicado para desplegar el código, los datos y la infraestructura.

Como la implementación sigue los mismos pasos que el despliegue en producción, también se ejecutarán las migraciones de bases de datos. Esta es una excelente manera de validar que las migraciones funcionen con los datos de producción.

Los entornos que no son ``master`` son muy similares al ``master`` anterior, excepto por algunas pequeñas diferencias: por ejemplo, los correos electrónicos no se envían por defecto.

.. index::
    single: Symfony CLI;cloud:url

Una vez finalizado el despliegue, abre la nueva rama en un navegador:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Ten en cuenta que todos los comandos de Upsun funcionan en la rama actual de Git. Este comando abre la URL que ha sido desplegada para la rama ``sessions-in-db``; la URL será algo similar a ``https://sessions-in-db-xxx.eu-5.platformsh.site/``.

Prueba el sitio web en este nuevo entorno, deberías ver todos los datos que creaste en el entorno *master*.

Si se añaden más conferencias en el entorno ``master``, no aparecerán en el entorno ``sessions-in-db`` y viceversa. Los entornos son independientes y aislados.

Si el código se desarrolla en *master*, siempre puedes hacer *rebase* a la rama de Git y desplegar la versión actualizada, resolviendo los conflictos tanto para el código como para la infraestructura.

.. index::
    single: Symfony CLI;cloud:env:sync

Incluso puedes sincronizar los datos desde la rama *master* con el entorno ``sessions-in-db``:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Depurando las implementaciones de producción antes del despliegue
------------------------------------------------------------------

.. index::
    single: Upsun;Debugging

Por defecto, todos los entornos Upsun utilizan la misma configuración que el entorno ``master``/``prod`` (también conocido como entorno Symfony ``prod``). Esto te permite probar la aplicación en condiciones reales. Te da la sensación de desarrollar y probar directamente en servidores de producción, pero sin los riesgos asociados con ello. Esto me recuerda a los buenos tiempos en los que estábamos realizando la implementación a través de FTP.

.. index::
    single: Symfony CLI;cloud:env:debug

En caso que tengas algún problema, es posible que quieras cambiar al entorno Symfony ``dev``:

.. code-block:: terminal

    $ symfony cloud:env:debug

Cuando hayas terminado, vuelve a los ajustes de producción:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    **Nunca** habilites el entorno ``dev`` y nunca habilites el *Symfony Profiler* en la rama ``master``; esto haría tu aplicación realmente lenta y abriría muchas vulnerabilidades serias de seguridad.

Probando las implementaciones de producción antes del despliegue
-----------------------------------------------------------------

Tener acceso a la próxima versión del sitio web con datos de producción te abre un abanico de oportunidades: desde pruebas visuales de regresión hasta pruebas de rendimiento. `Blackfire`_ es la herramienta perfecta para este trabajo.

Consulta el paso :doc:`Performance <29-performance>` para obtener más información sobre cómo puedes utilizar Blackfire para probar tu código antes del despliegue.

Fusionando en producción
-------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Cuando estés satisfecho con los cambios en la rama, vuelve a fusionar el código y la infraestructura con la rama master de Git:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

Y haz el despliegue:

.. code-block:: terminal

    $ symfony cloud:push

Cuando se despliega, sólo los cambios de código e infraestructura son enviados a Upsun; los datos no se ven afectados de ninguna manera.

Limpiando
---------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Finalmente, elimina la rama Git y el entorno Upsun:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Yendo más allá

    * `Ramificación en Git`_ ;

.. _`resto de partes interesadas`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Ramificación en Git`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
