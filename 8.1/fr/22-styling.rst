Styliser l'interface utilisateur
================================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

Nous n'avons pas consacré de temps au design de l'interface utilisateur. Pour la styliser comme un pro, nous utiliserons une stack moderne basée sur *AssetMapper*, le composant Symfony qui gère nos assets depuis la toute première étape de ce livre.

AssetMapper s'appuie sur les standards modernes du web : les fichiers JavaScript et CSS sont servis tels quels et reliés entre eux par une *importmap*, ce qui permet au navigateur de charger directement des *modules ES* natifs. Pas de bundler, pas d'étape de build, pas de Node.js.

Jetez un coup d'œil au fichier ``importmap.php`` à la racine du projet : il décrit les paquets JavaScript utilisés par l'application. La fonction Twig ``importmap()``, appelée dans ``templates/base.html.twig``, les expose au navigateur.

Tirer parti de Bootstrap
------------------------

.. index::
    single: Bootstrap

Pour commencer avec de bonnes bases et construire un site web responsive, un framework CSS comme `Bootstrap`_ peut faire gagner beaucoup de temps. Installez-le en tant que paquet importmap :

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

La commande enregistre le paquet dans ``importmap.php`` et le télécharge (ainsi que sa dépendance ``@popperjs/core``) dans ``assets/vendor/`` ; l'application ne dépend d'aucun CDN à l'exécution.

Importez Bootstrap dans le point d'entrée JavaScript principal (nous en avons aussi profité pour supprimer le message de bienvenue par défaut) :

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

Notez que ``app.css`` est importé *après* les styles de Bootstrap, afin que nos personnalisations l'emportent.

Le système de formulaire de Symfony supporte Bootstrap nativement avec un thème spécial. Activez-le :

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Styliser le HTML
----------------

Nous pouvons maintenant styliser l'application. Téléchargez et extrayez l'archive à la racine du projet :

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Jetez un coup d'œil aux templates, vous pourriez y apprendre une astuce ou deux à propos de Twig.

Servir les assets
-----------------

.. index::
    single: AssetMapper;asset-map:compile

Il n'y a rien à builder : rechargez une page et les changements sont visibles. En développement, AssetMapper sert directement les fichiers d'assets.

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

En production, Upsun exécute automatiquement la commande ``asset-map:compile`` pendant la phase de *build* : tous les assets sont copiés dans ``public/assets/`` avec un hash de version dans leur nom de fichier, ce qui permet un cache HTTP sûr et de longue durée.

.. sidebar:: Aller plus loin

    * La `documentation du composant AssetMapper`_ ;

    * La `spécification importmap`_ ;

    * La `documentation de Bootstrap`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`documentation du composant AssetMapper`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`spécification importmap`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`documentation de Bootstrap`: https://getbootstrap.com/docs/
