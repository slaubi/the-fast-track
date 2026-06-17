Dando estilo a la interfaz de usuario
=====================================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

No hemos dedicado tiempo al diseño de la interfaz de usuario. Para darle estilo como un profesional, usaremos una pila moderna basada en *AssetMapper*, el componente de Symfony que ha estado gestionando nuestros assets desde el primer paso de este libro.

AssetMapper adopta los estándares web modernos: los archivos JavaScript y CSS se sirven tal cual y se conectan entre sí mediante un *importmap*, dejando que el navegador cargue directamente los *módulos ES* nativos. Sin empaquetador, sin paso de compilación, sin Node.js.

Echa un vistazo al archivo ``importmap.php`` en la raíz del proyecto: describe los paquetes JavaScript que usa la aplicación. La función Twig ``importmap()`` que se llama en ``templates/base.html.twig`` los expone al navegador.

Aprovechando Bootstrap
----------------------

.. index::
    single: Bootstrap

Para empezar con buenos valores predeterminados y construir un sitio web responsive, un framework CSS como `Bootstrap`_ puede ayudarte mucho. Instálalo como un paquete del importmap:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

El comando registra el paquete en ``importmap.php`` y lo descarga (junto con su dependencia ``@popperjs/core``) en ``assets/vendor/``; la aplicación no depende de una CDN en tiempo de ejecución.

Importa Bootstrap en el punto de entrada principal de JavaScript (también hemos limpiado el mensaje de bienvenida predeterminado):

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -5,6 +5,6 @@ import './stimulus_bootstrap.js';
      * This file will be included onto the page via the importmap() Twig function,
      * which should already be in your base.html.twig.
      */
    +import 'bootstrap';
    +import 'bootstrap/dist/css/bootstrap.min.css';
     import './styles/app.css';
    -
    -console.log('This log comes from assets/app.js - welcome to AssetMapper! 🎉');

Fíjate en que ``app.css`` se importa *después* de los estilos de Bootstrap para que nuestras personalizaciones prevalezcan.

El sistema de formularios de Symfony soporta Bootstrap de forma nativa con un tema especial, habilítalo:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Dando estilo al HTML
--------------------

Ya estamos listos para dar estilo a la aplicación. Descarga y descomprime el archivo en la raíz del proyecto:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Echa un vistazo a las plantillas, puede que aprendas algún truco sobre Twig.

Sirviendo los assets
--------------------

.. index::
    single: AssetMapper;asset-map:compile

No hay nada que compilar: actualiza una página y los cambios se ven en vivo. En desarrollo, AssetMapper sirve los archivos de los assets directamente.

Tómate tu tiempo para descubrir los cambios visuales. Echa un vistazo al nuevo diseño en un navegador.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

El formulario de inicio de sesión generado también tiene estilo ahora, ya que el *bundle* Maker usa las clases CSS de Bootstrap de forma predeterminada:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

Para producción, Upsun ejecuta automáticamente el comando ``asset-map:compile`` durante la fase de compilación: todos los assets se copian a ``public/assets/`` con un hash de versión en sus nombres de archivo, lo que permite un almacenamiento en caché HTTP seguro y de larga duración.

.. sidebar:: Yendo más allá

    * La `documentación del componente AssetMapper`_ ;

    * La `especificación de importmap`_ ;

    * La `documentación de Bootstrap`_ .

.. _`Bootstrap`: https://getbootstrap.com/
.. _`documentación del componente AssetMapper`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`especificación de importmap`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`documentación de Bootstrap`: https://getbootstrap.com/docs/
