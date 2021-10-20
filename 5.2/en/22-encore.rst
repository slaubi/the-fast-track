Styling the User Interface with Webpack
=======================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

We have spent no time on the design of the user interface. To style like a pro, we will use a modern stack, based on `Webpack <https://webpack.js.org/>`_. And to add a Symfony touch and ease its integration with the application, let's install *Webpack Encore*:

.. code-block:: bash

    $ symfony composer req encore

A full Webpack environment has been created for you: ``package.json`` and ``webpack.config.js`` have been generated and contain good default configuration. Open ``webpack.config.js``, it uses the Encore abstraction to configure Webpack.

The ``package.json`` file defines some nice commands that we will use all the time.

The ``assets`` directory contains the main entry points for the project assets: ``styles/app.css`` and ``app.js``.

Using Sass
----------

.. index::
    single: Sass

Instead of using plain CSS, let's switch to `Sass <https://sass-lang.com/>`_:

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

     // start the Stimulus application
     import './bootstrap';

Install the Sass loader:

.. code-block:: bash

    $ yarn add node-sass sass-loader --dev

And enable the Sass loader in webpack:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -56,7 +56,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

How did I know which packages to install? If we had tried to build our assets without them, Encore would have given us a nice error message suggesting the ``yarn add`` command needed to install dependencies to load ``.scss`` files.

Leveraging Bootstrap
--------------------

.. index::
    single: Bootstrap

To start with good defaults and build a responsive website, a CSS framework like `Bootstrap <https://getbootstrap.com/>`_ can go a long way. Install it as a package:

.. code-block:: bash

    $ yarn add bootstrap@4 jquery popper.js bs-custom-file-input --dev

Require Bootstrap in the CSS file (we have also cleaned up the file):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Do the same for the JS file:

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

The Symfony form system supports Bootstrap natively with a special theme, enable it:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_4_layout.html.twig']

Styling the HTML
----------------

We are now ready to style the application. Download and expand the archive at the root of the project:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-5.2.zip', 'guestbook-5.2.zip');"
    $ unzip -o guestbook-5.2.zip
    $ rm guestbook-5.2.zip

Have a look at the templates, you might learn a trick or two about Twig.

Building Assets
---------------

.. index::
    single: Symfony CLI;run

One major change when using Webpack is that CSS and JS files are not usable directly by the application. They need to be "compiled" first.

In development, compiling the assets can be done via the ``encore dev`` command:

.. code-block:: bash

    $ symfony run yarn encore dev

Instead of executing the command each time there is a change, send it to the background and let it watch JS and CSS changes:

.. code-block:: bash
    :class: ignore

    $ symfony run -d yarn encore dev --watch

Take the time to discover the visual changes. Have a look at the new design in a browser.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

The generated login form is now styled as well as the Maker bundle uses Bootstrap CSS classes by default:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

For production, SymfonyCloud automatically detects that you are using Encore and compiles the assets for you during the build phase.

.. sidebar:: Going Further

    * `Webpack docs <https://webpack.js.org/concepts/>`_;

    * `Symfony Webpack Encore docs <https://symfony.com/doc/current/frontend.html>`_;

    * `SymfonyCasts Webpack Encore tutorial <https://symfonycasts.com/screencast/webpack-encore>`_.
