De gebruikersinterface stylen
=============================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

We hebben nog geen tijd besteed aan het ontwerp van de gebruikersinterface. Om als een pro te stylen, gebruiken we een moderne stack gebaseerd op *AssetMapper*, de Symfony-component die onze assets al vanaf de allereerste stap van dit boek beheert.

AssetMapper omarmt moderne webstandaarden: JavaScript- en CSS-bestanden worden ongewijzigd geserveerd en aan elkaar gekoppeld met een *importmap*, waardoor de browser native *ES modules* rechtstreeks kan laden. Geen bundler, geen build-stap, geen Node.js.

Bekijk het ``importmap.php`` bestand in de root van het project: het beschrijft de JavaScript-packages die door de applicatie worden gebruikt. De ``importmap()`` Twig-functie die in ``templates/base.html.twig`` wordt aangeroepen, stelt ze beschikbaar aan de browser.

Bootstrap inzetten
------------------

.. index::
    single: Bootstrap

Om met goede standaardinstellingen te beginnen en een responsieve website te bouwen, kan een CSS-framework zoals `Bootstrap`_ een groot verschil maken. Installeer het als een importmap-package:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

Het command registreert de package in ``importmap.php`` en downloadt deze (en de ``@popperjs/core`` afhankelijkheid) naar ``assets/vendor/``; de applicatie is tijdens runtime niet afhankelijk van een CDN.

Importeer Bootstrap in het belangrijkste JavaScript-entrypoint (we hebben ook het standaard welkomstbericht opgeschoond):

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

Merk op dat ``app.css`` *na* de Bootstrap-styles wordt geïmporteerd, zodat onze aanpassingen winnen.

Het Symfony form-systeem ondersteunt Bootstrap native met een speciaal thema, schakel het in:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

De HTML stylen
--------------

We zijn nu klaar om de applicatie te stylen. Download het archief en pak het uit in de root van het project:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Bekijk de templates, je leert misschien een trucje of twee over Twig.

De assets serveren
------------------

.. index::
    single: AssetMapper;asset-map:compile

Er is niets te builden: ververs een pagina en de wijzigingen zijn live. In development serveert AssetMapper de assetbestanden rechtstreeks.

Neem de tijd om de visuele veranderingen te ontdekken. Bekijk het nieuwe ontwerp in een browser.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Het gegenereerde inlogformulier is nu ook gestyled, aangezien de Maker bundle standaard Bootstrap CSS-classes gebruikt:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

Voor productie voert Upsun automatisch het ``asset-map:compile`` command uit tijdens de buildfase: alle assets worden gekopieerd naar ``public/assets/`` met een versie-hash in hun bestandsnamen, wat veilige, langdurige HTTP-caching mogelijk maakt.

.. sidebar:: Verder gaan

    * De `AssetMapper component docs`_;

    * De `importmap specification`_;

    * De `Bootstrap docs`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`AssetMapper component docs`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`importmap specification`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`Bootstrap docs`: https://getbootstrap.com/docs/
