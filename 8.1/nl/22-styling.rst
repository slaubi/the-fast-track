De gebruikersinterface stylen met Webpack
=========================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

We hebben geen tijd besteed aan het ontwerp van de gebruikersinterface. Om als een pro te stylen, gebruiken we een moderne stack, gebaseerd op `Webpack`_. En om een snufje Symfony toe te voegen en de integratie met de toepassing te vergemakkelijken, installeren we *Webpack Encore*:

.. code-block:: terminal

    $ symfony composer req encore

Er is een volledige Webpack-omgeving voor je gecreëerd: ``package.json`` en ``webpack.config.js`` zijn gegenereerd en bevatten een goede standaardconfiguratie. Open ``webpack.config.js``, het gebruikt de Encore abstractie om Webpack te configureren.

Het ``package.json``-bestand definieert een aantal handige commando's die we vaak zullen gaan gebruiken.

De ``assets``-map bevat de belangrijkste toegangspunten voor de assets van het project: ``styles/app.css`` en ``app.js``.

Sass gebruiken
--------------

.. index::
    single: Sass

Laten we overschakelen naar `Sass`_, in plaats van gewone CSS te gebruiken:

.. code-block:: terminal

    $ mv assets/styles/app.css assets/styles/app.scss

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -6,7 +6,7 @@
      */

     // any CSS you import will output into a single css file (app.css in this case)
    -import './styles/app.css';
    +import './styles/app.scss';

     // start the Stimulus application
     import './bootstrap';

Installeer de Sass-loader:

.. code-block:: terminal

    $ npm install node-sass sass-loader --save-dev

En schakel in webpack de Sass-loader in:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -57,7 +57,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

Hoe wist ik welke packages ik moest installeren? Als we hadden geprobeerd om onze assets te bouwen zonder de juiste packages, zou Encore ons een mooie foutmelding hebben gegeven met de suggestie het commando ``npm install`` te draaien. Daarmee installeer je de dependencies die benodigd zijn om ``.scss``-bestanden te laden.

Bootstrap gebruiken
-------------------

.. index::
    single: Bootstrap

Om te beginnen met goede standaardinstellingen en het bouwen van een responsive website, kom je met een CSS-framework als `Bootstrap`_ al een heel eind. Installeer het als een package:

.. code-block:: terminal

    $ npm install bootstrap @popperjs/core bs-custom-file-input --save-dev

Voeg Bootstrap toe aan het CSS-bestand (we hebben ook het bestand opgeschoond):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Doe hetzelfde voor het JS-bestand:

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -7,6 +7,10 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';

     // start the Stimulus application
     import './bootstrap';
    +
    +bsCustomFileInput.init();

Het Symfony-formuliersysteem ondersteunt Bootstrap standaard met een speciaal thema. Schakel deze in:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Styling toevoegen aan de HTML
-----------------------------

We zijn nu klaar om de applicatie te stylen. Download de zip in de root van het project en pak het uit:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-6.2.zip', 'guestbook-6.2.zip');"
    $ unzip -o guestbook-6.2.zip
    $ rm guestbook-6.2.zip

Neem eens een kijkje in de templates, misschien leer je nog een paar handige Twig-trucjes.

Assets genereren
----------------

.. index::
    single: Symfony CLI;run

Wat anders is bij Webpack, is dat CSS- en JS-bestanden niet direct gebruikt worden door de toepassing. Ze moeten eerst "gecompileerd" worden.

Tijdens development kun je de assets compileren met het ``encore dev``-commando:

.. code-block:: terminal

    $ symfony run npm run dev

In plaats van het commando uit te voeren elke keer als er een wijziging is, kun je het ook in de achtergrond je JS en CSS automatisch in de gaten laten houden:

.. code-block:: terminal
    :class: ignore

    $ symfony run -d npm run watch

Neem de tijd om de visuele veranderingen te ontdekken. Bekijk het nieuwe ontwerp in een browser.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Het gegenereerde aanmeldingsformulier is nu gestyled en de Maker-bundle maakt standaard gebruik van Bootstrap CSS-classes:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

Voor productie detecteert Upsun automatisch dat je Encore gebruikt en stelt de assets voor je samen tijdens het genereren.

.. sidebar:: Verder gaan

    * `Webpack documentatie`_;

    * `Symfony Webpack Encore documentatie`_;

    * `SymfonyCasts Webpack Encore-tutorial`_.

.. _`Webpack`: https://webpack.js.org/
.. _`Sass`: https://sass-lang.com/
.. _`Bootstrap`: https://getbootstrap.com/
.. _`Webpack documentatie`: https://webpack.js.org/concepts/
.. _`Symfony Webpack Encore documentatie`: https://symfony.com/doc/current/frontend.html
.. _`SymfonyCasts Webpack Encore-tutorial`: https://symfonycasts.com/screencast/webpack-encore
