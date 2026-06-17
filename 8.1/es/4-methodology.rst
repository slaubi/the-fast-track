Adoptando una metodología
==========================

Enseñar consiste en repetir lo mismo una y otra vez. No voy a hacer eso. Te lo prometo. Al final de cada paso, deberías marcarte un baile y guardar tu trabajo. Es como ``Ctrl+S`` pero para un sitio web.

Implementando una estrategia de Git
-----------------------------------

.. index::
    single: Git;add
    single: Git;commit

No olvides hacer un *commit* con los cambios cuando acabes cada paso:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Puedes agregar "todo" de forma segura ya que Symfony administra un fichero ``.gitignore`` por ti. Y cada paquete puede añadir más configuraciones. Échale un vistazo al contenido actual:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Las cadenas de caracteres raros son marcadores añadidos por Symfony Flex para que sepas qué eliminar si decides desinstalar una dependencia. Te lo dije, todo el trabajo tedioso lo hace Symfony, no tú.

Podría ser bueno hacer *push* de tu repositorio a algún servidor externo. GitHub, GitLab o Bitbucket son buenas opciones.

Si estás desplegando en Upsun, ya tienes una copia del repositorio Git, pero no deberías depender exclusivamente de él. Se usa solamente para el despliegue. No es una copia de seguridad.

Despliegue continuo a producción
---------------------------------

.. index::
    single: Symfony CLI;deploy

Otra buena costumbre es desplegar con frecuencia. Hacerlo al final de cada paso podría considerarse un ritmo adecuado.

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
