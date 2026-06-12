Stilizzare l'interfaccia utente
===============================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

Non abbiamo dedicato tempo al design dell'interfaccia utente. Per stilizzare come professionisti, useremo uno stack moderno basato su *AssetMapper*, il componente Symfony che gestisce i nostri asset fin dal primissimo passo di questo libro.

AssetMapper abbraccia gli standard moderni del web: i file JavaScript e CSS vengono serviti così come sono e collegati tra loro da una *importmap*, lasciando che il browser carichi direttamente i *moduli ES* nativi. Nessun bundler, nessuna fase di build, niente Node.js.

Dare un'occhiata al file ``importmap.php`` nella cartella principale del progetto: descrive i pacchetti JavaScript usati dall'applicazione. La funzione Twig ``importmap()``, richiamata in ``templates/base.html.twig``, li espone al browser.

Sfruttare Bootstrap
-------------------

.. index::
    single: Bootstrap

Per iniziare con buone impostazioni predefinite e costruire un sito web responsive, un framework CSS come `Bootstrap`_ può fare molto. Installarlo come pacchetto importmap:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

Il comando registra il pacchetto in ``importmap.php`` e lo scarica (insieme alla sua dipendenza ``@popperjs/core``) in ``assets/vendor/``; l'applicazione non dipende da alcun CDN in fase di esecuzione.

Importare Bootstrap nel punto di ingresso JavaScript principale (abbiamo anche ripulito il messaggio di benvenuto predefinito):

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

Si noti che ``app.css`` viene importato *dopo* gli stili di Bootstrap, in modo che le nostre personalizzazioni abbiano la precedenza.

Il sistema dei form di Symfony supporta Bootstrap nativamente con un tema speciale, abilitiamolo:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Stilizzare l'HTML
-----------------

Ora siamo pronti per impostare lo stile dell'applicazione. Scarichiamo ed estraiamo l'archivio nella cartella principale del progetto:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Date un'occhiata ai template, potreste imparare un paio di trucchi su Twig.

Servire gli asset
-----------------

.. index::
    single: AssetMapper;asset-map:compile

Non c'è nulla da compilare: ricaricare una pagina e le modifiche sono già attive. In sviluppo, AssetMapper serve direttamente i file degli asset.

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

In produzione, Upsun esegue automaticamente il comando ``asset-map:compile`` durante la fase di build: tutti gli asset vengono copiati in ``public/assets/`` con un hash di versione nel nome del file, abilitando una cache HTTP sicura e di lunga durata.

.. sidebar:: Andare oltre

    * La `documentazione del componente AssetMapper`_;

    * La `specifica importmap`_;

    * La `documentazione di Bootstrap`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`documentazione del componente AssetMapper`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`specifica importmap`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`documentazione di Bootstrap`: https://getbootstrap.com/docs/
