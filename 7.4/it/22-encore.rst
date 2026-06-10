Impostare lo stile dell'interfaccia utente con Webpack
======================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

Non abbiamo speso tempo nella progettazione dell'interfaccia utente. Per impostare uno stile professionale, utilizzeremo uno stack moderno basato su `Webpack`_, e per aggiungere un tocco di Symfony e facilitare la sua integrazione con l'applicazione, installiamo *Webpack Encore*:

.. code-block:: terminal

    $ symfony composer rem asset-mapper
    $ symfony composer req encore

Un ambiente Webpack completo è stato creato per noi: ``package.json`` e ``webpack.config.js`` sono stati generati e contengono una buona configurazione predefinita. Aprendo ``webpack.config.js``, potremo notare l'utilizzo dell'astrazione Encore usata per configurare Webpack.

Il file ``package.json`` definisce alcuni comandi che useremo sempre.

La cartella ``assets``contiene i principali punti di ingresso per le risorse del progetto: ``styles/app.css`` e ``app.js``.

Utilizzo di Sass
----------------

.. index::
    single: Sass

Invece di usare i normali CSS, passiamo a `Sass`_:

.. code-block:: terminal

    $ mv assets/styles/app.css assets/styles/app.scss

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -6,4 +6,4 @@
      */

     // any CSS you import will output into a single css file (app.css in this case)
    -import './styles/app.css';
    +import './styles/app.scss';

Installare sass-loader:

.. code-block:: terminal

    $ npm install sass sass-loader --save-dev

E abilitare il Sass loader in webpack:

.. code-block:: diff
    :caption: patch_file

    --- i/webpack.config.js
    +++ w/webpack.config.js
    @@ -57,7 +57,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

Come facevo a sapere quali pacchetti installare? Se si tenta un build senza pacchetti, Encore mostra un messaggio di errore e suggerisce il comando ``npm install`` per installare le dipendenze (necessarie per caricare i file ``.scss``).

Sfruttare Bootstrap
-------------------

.. index::
    single: Bootstrap

Per iniziare con buone impostazioni predefinite e costruire un sito responsive, un framework CSS come `Bootstrap`_ può aiutarci molto. Installiamolo come pacchetto:

.. code-block:: terminal

    $ npm install bootstrap @popperjs/core bs-custom-file-input --save-dev

Richiediamo Bootstrap nel file CSS (abbiamo anche ripulito il file):

.. code-block:: diff
    :caption: patch_file

    --- i/assets/styles/app.scss
    +++ w/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Facciamo lo stesso per il file JS:

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -7,3 +7,7 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';
    +
    +bsCustomFileInput.init();

Il sistema dei form di Symfony supporta Bootstrap nativamente con un tema speciale, abilitiamolo:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Impostare lo stile dell'HTML
----------------------------

Ora siamo pronti per impostare lo stile dell'applicazione. Scarichiamo ed estraiamo l'archivio nella cartella principale del progetto:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-7.4.zip', 'guestbook-7.4.zip');"
    $ unzip -o guestbook-7.4.zip
    $ rm guestbook-7.4.zip

Dando un'occhiata ai template, potreste imparare un paio di trucchi su Twig.

Build degli asset
-----------------

.. index::
    single: Symfony CLI;run

Un cambiamento importante quando si usa Webpack è che i file CSS e JS non sono utilizzabili direttamente dall'applicazione. Devono essere prima "compilati".

Durante lo sviluppo, gli asset possono essere compilati tramite il comando ``encore dev``:

.. code-block:: terminal

    $ symfony run npm run dev

Invece di eseguire il comando ogni volta che c'è una modifica, metterlo in background e lasciare che intercetti le modifiche dei file JS e CSS:

.. code-block:: terminal
    :class: ignore

    $ symfony run -d npm run watch

Prendetevi il tempo per scoprire i cambiamenti visivi. Date un'occhiata al nuovo design in un browser.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Il form di login generato ha ora uno stile, perché MakerBundle usa di default le classi CSS di Bootstrap:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

In produzione, Platform.sh rileva automaticamente che si sta usando Encore e compila gli asset per noi durante la fase di build.

.. sidebar:: Andare oltre

    * `Documentazione di Webpack`_;

    * `Documentazione di Webpack Encore`_;

    * `Guida a Webpack Encore su SymfonyCasts`_.

.. _`Webpack`: https://webpack.js.org/
.. _`Sass`: https://sass-lang.com/
.. _`Bootstrap`: https://getbootstrap.com/
.. _`Documentazione di Webpack`: https://webpack.js.org/concepts/
.. _`Documentazione di Webpack Encore`: https://symfony.com/doc/current/frontend.html
.. _`Guida a Webpack Encore su SymfonyCasts`: https://symfonycasts.com/screencast/webpack-encore
