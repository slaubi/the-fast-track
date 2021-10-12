Styliser l'interface avec Webpack
=================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

Nous n'avons pas encore travaillé sur la conception de l'interface. Pour styliser comme un pro, nous utiliserons une stack moderne, basée sur `Webpack <https://webpack.js.org/>`_. Et pour ajouter une touche de Symfony et faciliter son intégration avec l'application, installons *Webpack Encore* :

.. code-block:: bash

    $ symfony composer req encore

Un environnement Webpack complet a été créé pour vous : les fichiers ``package.json`` et ``webpack.config.js`` ont été générés avec une bonne configuration par défaut. Ouvrez ``webpack.config.js``, il utilise l'abstraction Encore pour configurer Webpack.

Le fichier ``package.json`` définit des commandes très pratiques que nous utiliserons sans arrêt.

Le répertoire ``assets`` contient les principaux points d'entrée des *assets* du projet : ``styles/app.css`` et ``app.js``.

Utiliser Sass
-------------

.. index::
    single: Sass

Au lieu de se contenter simplement de CSS, nous utiliserons `Sass <https://sass-lang.com/>`_ :

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

Installez le *loader* Sass :

.. code-block:: bash

    $ yarn add node-sass sass-loader --dev

Et activez-le dans webpack :

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

Comment savoir quels paquets installer ? Si nous avions essayé de générer nos *assets* sans ceux-ci, Encore nous aurait donné un joli message d'erreur suggérant la commande ``yarn add`` à exécuter pour installer les dépendances servant à charger les fichiers ``.scss``.

Tirer parti de Bootstrap
------------------------

.. index::
    single: Bootstrap

Pour commencer avec de bonnes bases et construire un site web *responsive*, un framework CSS comme `Bootstrap <https://getbootstrap.com/>`_ sera très utile. Installez-le comme paquet :

.. code-block:: bash

    $ yarn add bootstrap jquery popper.js bs-custom-file-input --dev

Ajoutez Bootstrap dans le fichier CSS (nous avons aussi nettoyé le fichier) :

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Faites de même pour le fichier JS :

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

Le système de formulaire de Symfony supporte Bootstrap nativement avec un thème spécial. Activez-le :

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_4_layout.html.twig']

Styliser le HTML
----------------

Nous pouvons maintenant styliser l'application. Téléchargez et extrayez l'archive à la racine du projet :

.. code-block:: bash

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-5.0.zip', 'guestbook-5.0.zip');"
    $ unzip -o guestbook-5.0.zip
    $ rm guestbook-5.0.zip

Jetez un coup d’œil aux templates, vous pourriez apprendre un truc ou deux sur Twig.

Générer les assets
--------------------

.. index::
    single: Symfony CLI;run

Il y a une différence majeure quand on utilise Webpack : les fichiers CSS et JS ne sont pas utilisables directement par l'application. Ils doivent d'abord être "compilés".

En développement, la compilation des ressources peut se faire avec la commande ``encore dev`` :

.. code-block:: bash

    $ symfony run yarn encore dev

Au lieu d'exécuter la commande chaque fois qu'il y a une modification, exécutez-la en arrière-plan et laissez-la surveiller les changements des fichiers JS et CSS :

.. code-block:: bash
    :class: ignore

    $ symfony run -d yarn encore dev --watch

Prenez le temps de découvrir les modifications visuelles. Jetez un coup d'œil au nouveau design dans un navigateur.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Étant donné que le *Maker Bundle* utilise les classes CSS de Bootstrap par défaut, le formulaire de connexion précédemment généré est maintenant automatiquement stylisé :

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

En production, SymfonyCloud détecte automatiquement que vous utilisez Encore et compile les ressources pour vous pendant la phase de *build*.

.. sidebar:: Aller plus loin

    * `Documentation Webpack <https://webpack.js.org/concepts/>`_ ;

    * `Documentation Symfony Webpack Encore <https://symfony.com/doc/current/frontend.html>`_ ;

    * `Tutoriel SymfonyCasts sur Webpack Encore <https://symfonycasts.com/screencast/webpack-encore>`_.
