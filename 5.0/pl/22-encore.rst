Stylowanie interfejsu użytkownika z wykorzystaniem narzędzia Webpack
======================================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

Nie poświęciliśmy zbyt wiele czasu na projektowanie interfejsu użytkownika. Aby go profesjonalnie ostylować, użyjemy nowoczesnego zestawu narzędzi, opartego na `Webpacku <https://webpack.js.org/>`_. Aby ułatwić jego integrację z naszą aplikacją zainstalujemy *Webpack Encore*:

.. code-block:: bash

    $ symfony composer req encore

Pełne środowisko Webpack zostało przygotowane: ``package.json`` i ``webpack.config.js`` zostały wygenerowane i zawierają dobrą domyślną konfigurację. Otwórz plik ``webpack.config.js`` – używa on abstrakcji Encore do konfiguracji Webpacka.

Plik ``package.json`` definiuje kilka ciekawych poleceń, których będziemy używać przez cały czas.

Katalog ``assets`` zawiera główne punkty wejścia dla zasobów projektu: ``styles/app.css`` oraz ``app.js``.

Używanie Sass
--------------

.. index::
    single: Sass

Zamiast używać zwykłego CSS, skorzystajmy z `Sass <https://sass-lang.com/>`_:

.. code-block:: bash

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

     // Need jQuery? Install it with "yarn add jquery", then uncomment to import it.
     // import $ from 'jquery';

Zainstaluj moduł ładowania Sass:

.. code-block:: bash

    $ yarn add node-sass sass-loader --dev

I uruchom go w Webpacku:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -54,7 +54,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

Skąd wiedziałem, które pakiety zainstalować? Gdybyśmy spróbowali zbudować nasze zasoby bez nich, Encore wyświetliłby komunikat błędu sugerujący uruchomienie polecenia ``yarn add`` potrzebnego do instalacji zależności służących do ładowania plików ``.scss``.

Wykorzystanie Bootstrapa
------------------------

.. index::
    single: Bootstrap

Potrzebujemy solidnych podstaw do zbudowania responsywnej strony internetowej. Framework CSS, taki jak `Bootstrap <https://getbootstrap.com/>`_ spełnia te warunki. Zainstaluj go jako pakiet (ang. package):

.. code-block:: bash

    $ yarn add bootstrap jquery popper.js bs-custom-file-input --dev

Dołącz (ang. require) Bootstrapa w pliku CSS (wyczyściliśmy również ten plik):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Zrób to samo w przypadku pliku JS:

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -7,8 +7,7 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';

    -// Need jQuery? Install it with "yarn add jquery", then uncomment to import it.
    -// import $ from 'jquery';
    -
    -console.log('Hello Webpack Encore! Edit me in assets/app.js');
    +bsCustomFileInput.init();

System formularzy Symfony obsługuje natywnie Bootstrapa ze specjalnym motywem, włącz go:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_4_layout.html.twig']

Stylowanie HTML
---------------

Jesteśmy teraz gotowi do stylowania aplikacji. Pobierz i rozpakuj archiwum w katalogu głównym projektu:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-5.0.zip', 'guestbook-5.0.zip');"
    $ unzip -o guestbook-5.0.zip
    $ rm guestbook-5.0.zip

Spójrz na szablony Twig, znajdziesz w nich kilka ciekawych rozwiązań.

Budowanie zasobów (ang. assets)
--------------------------------

.. index::
    single: Symfony CLI;run

Jedną z głównych różnic w korzystaniu z Webpacka jest to, że pliki CSS i JS nie mogą być wykorzystywane bezpośrednio przez aplikację. Najpierw trzeba je "skompilować".

Podczas rozwoju aplikacji, kompilacja zasobów może być wykonana za pomocą polecenia ``encore dev``:

.. code-block:: bash

    $ symfony run yarn encore dev

Zamiast wykonywać polecenie za każdym razem, gdy wystąpi zmiana, uruchom je w tle i pozwól mu obserwować zmiany w plikach JS i CSS:

.. code-block:: bash
    :class: ignore

    $ symfony run -d yarn encore dev --watch

Poświęć czas na odkrycie zmian wizualnych. Przyjrzyj się nowemu wyglądowi w przeglądarce.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Wygenerowany formularz logowania jest teraz ostylowany, a Maker Bundle używa domyślnie klas CSS Bootstrapa:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

W środowisku produkcyjnym SymfonyCloud automatycznie wykrywa, że używasz Encore i kompiluje zasoby za Ciebie podczas fazy budowania aplikacji.

.. sidebar:: Idąc dalej

    * `Dokumentacja Webpack <https://webpack.js.org/concepts/>`_;

    * `Dokumentacja Symfony Webpack Encore <https://symfony.com/doc/current/frontend.html>`_;

    * `Samouczek SymfonyCasts Webpack Encore <https://symfonycasts.com/screencast/webpack-encore>`_.
