Créer une SPA (Single Page Application)
=======================================

.. index::
    single: SPA
    single: Mobile

La plupart des commentaires seront soumis pendant les conférences, et certaines personnes n'y apportent pas leur ordinateur portable. Mais ils auront probablement leur téléphone. Pourquoi ne pas créer une application mobile permettant de lire rapidement les commentaires de la conférence ?

Une façon de créer une telle application mobile est de créer une Single Page Application (SPA) Javascript. Une SPA s'exécute localement, a accès au stockage local, peut faire des appels à des API HTTP distantes et peut s'appuyer sur les *service workers* pour créer une expérience presque native.

Créer l'application
-------------------

Pour créer l'application mobile, nous allons utiliser `Preact`_ et **Symfony Encore**. **Preact** est une petite base efficace convenant parfaitement à la SPA du livre d'or.

Afin de rendre le site web et la SPA uniforme, nous allons réutiliser les feuilles de style Sass du site web pour l'application mobile.

Créez la SPA dans le répertoire ``spa`` et copiez les feuilles de style du site web :

.. code-block:: bash

    $ mkdir -p spa/src spa/public spa/assets/styles
    $ cp assets/styles/*.scss spa/assets/styles/
    $ cd spa

.. note::

    Nous avons créé un répertoire ``public`` car nous allons principalement interagir avec la SPA via un navigateur. Nous aurions pu le nommer ``build`` si nous avions voulu nous limiter à une application mobile.

Pour bien faire les choses, créez un fichier ``.gitignore`` :

.. code-block:: text
    :caption: .gitignore

    /node_modules/
    /public/
    /npm-debug.log
    /yarn-error.log
    # used later by Cordova
    /app/

Initialize the ``package.json`` file (equivalent of the ``composer.json``
file for JavaScript):

.. code-block:: bash

    $ yarn init -y

Maintenant, ajoutez quelques dépendances requises :

.. code-block:: bash

    $ yarn add @symfony/webpack-encore @babel/core @babel/preset-env babel-preset-preact preact html-webpack-plugin bootstrap

La dernière étape de configuration consiste à créer la configuration Webpack Encore :

.. code-block:: javascript
    :caption: webpack.config.js
    :emphasize-lines: 8,11

    const Encore = require('@symfony/webpack-encore');
    const HtmlWebpackPlugin = require('html-webpack-plugin');

    Encore
        .setOutputPath('public/')
        .setPublicPath('/')
        .cleanupOutputBeforeBuild()
        .addEntry('app', './src/app.js')
        .enablePreactPreset()
        .enableSingleRuntimeChunk()
        .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
    ;

    module.exports = Encore.getWebpackConfig();

Créer le template principal de la SPA
-------------------------------------

Il est temps de créer le template initial dans lequel Preact fera le rendu de l'application :

.. code-block:: html
    :caption: src/index.ejs
    :emphasize-lines: 12

    <!DOCTYPE html>
    <html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <meta name="msapplication-tap-highlight" content="no" />
        <meta name="viewport" content="user-scalable=no, initial-scale=1, maximum-scale=1, minimum-scale=1, width=device-width" />

        <title>Conference Guestbook application</title>
    </head>
    <body>
        <div id="app"></div>
    </body>
    </html>

JavaScript générera le rendu de l'application dans la balise ``<div>``. Voici la première version du code qui affichera la vue "Hello World" :

.. code-block:: text
    :caption: src/app.js
    :emphasize-lines: 3,11

    import {h, render} from 'preact';

    function App() {
        return (
            <div>
                Hello world!
            </div>
        )
    }

    render(<App />, document.getElementById('app'));

La dernière ligne enregistre la fonction ``App()`` sur l'élément ``#app`` de la page HTML.

Maintenant, tout est prêt !

Exécuter la SPA dans le navigateur web
--------------------------------------

.. index::
    single: Symfony CLI;server:start
    single: Symfony CLI;server:stop

Comme cette application est indépendante du site web principal, nous avons besoin d'un autre serveur web :

.. code-block:: bash
    :class: hide

    $ symfony server:stop

.. code-block:: bash

    $ symfony server:start -d --passthru=index.html

L'option ``--passthru`` indique au serveur web de transmettre toutes les requêtes HTTP au fichier ``public/index.html`` (``public/`` est le répertoire racine par défaut du serveur web). Cette page est gérée par l'application Preact et récupère la page à afficher via l'historique du "navigateur".

Pour compiler les fichiers CSS **et JavaScript**, exécutez ``yarn`` :

.. code-block:: bash

    $ yarn encore dev

Ouvrez la SPA dans un navigateur :

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Et contemplez notre SPA hello world :

.. figure:: screenshots/spa.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Ajouter un routeur pour gérer les états
---------------------------------------

La SPA n'est actuellement pas en mesure de traiter plusieurs pages. Pour pouvoir les implémenter, nous avons besoin d'un routeur, comme pour Symfony. Nous allons utiliser **preact-router**. Il prend une URL en entrée et la fait correspondre à un composant Preact à afficher.

Installez preact-router :

.. code-block:: bash

    $ yarn add preact-router

Créez une page pour l'accueil (un *composant Preact*) :

.. code-block:: text
    :caption: src/pages/home.js

    import {h} from 'preact';

    export default function Home() {
        return (
            <div>Home</div>
        );
    };

Et une autre pour la page d'une conférence :

.. code-block:: text
    :caption: src/pages/conference.js

    import {h} from 'preact';

    export default function Conference() {
        return (
            <div>Conference</div>
        );
    };

Remplacez le ``div`` "Hello World" par le composant ``Router`` :

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 15,17,20-23

    --- a/src/app.js
    +++ b/src/app.js
    @@ -1,9 +1,22 @@
     import {h, render} from 'preact';
    +import {Router, Link} from 'preact-router';
    +
    +import Home from './pages/home';
    +import Conference from './pages/conference';

     function App() {
         return (
             <div>
    -            Hello world!
    +            <header>
    +                <Link href="/">Home</Link>
    +                <br />
    +                <Link href="/conference/amsterdam2019">Amsterdam 2019</Link>
    +            </header>
    +
    +            <Router>
    +                <Home path="/" />
    +                <Conference path="/conference/:slug" />
    +            </Router>
             </div>
         )
     }

Rebuildez l'application :

.. code-block:: bash

    $ yarn encore dev

Si vous rafraîchissez l'application dans le navigateur, vous pouvez maintenant cliquer sur les liens "Accueil" et conférence. Notez que l'URL du navigateur et les boutons précédent/suivant de votre navigateur fonctionnent normalement.

Styliser la SPA
---------------

Comme pour le site web, ajoutons le loader Sass :

.. code-block:: bash

    $ yarn add node-sass sass-loader

Activez le loader Sass dans Webpack et ajoutez une référence à la feuille de style :

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -1,3 +1,5 @@
    +import '../assets/styles/app.scss';
    +
     import {h, render} from 'preact';
     import {Router, Link} from 'preact-router';

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -7,6 +7,7 @@ Encore
         .cleanupOutputBeforeBuild()
         .addEntry('app', './src/app.js')
         .enablePreactPreset()
    +    .enableSassLoader()
         .enableSingleRuntimeChunk()
         .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
     ;

Nous pouvons désormais mettre à jour l'application pour utiliser les feuilles de style :

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -9,10 +9,20 @@ import Conference from './pages/conference';
     function App() {
         return (
             <div>
    -            <header>
    -                <Link href="/">Home</Link>
    -                <br />
    -                <Link href="/conference/amsterdam2019">Amsterdam 2019</Link>
    +            <header className="header">
    +                <nav className="navbar navbar-light bg-light">
    +                    <div className="container">
    +                        <Link className="navbar-brand mr-4 pr-2" href="/">
    +                            &#128217; Guestbook
    +                        </Link>
    +                    </div>
    +                </nav>
    +
    +                <nav className="bg-light border-bottom text-center">
    +                    <Link className="nav-conference" href="/conference/amsterdam2019">
    +                        Amsterdam 2019
    +                    </Link>
    +                </nav>
                 </header>

                 <Router>

Rebuildez encore l'application :

.. code-block:: bash

    $ yarn encore dev

Vous pouvez à présent profiter d'une SPA entièrement stylisée :

.. figure:: screenshots/spa-home.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Récupérer les données depuis l'API
----------------------------------

La structure de l'application Preact est maintenant terminée : Preact Router gère les états de la page, incluant le slug des conférences dans l'URL, et la feuille de style principale de l'application est utilisée pour styliser la SPA.

Pour rendre la SPA dynamique, nous avons besoin de récupérer les données de l'API via des appels HTTP.

Configurez Webpack pour exposer la variable d'environnement contenant le point d'entrée de l'API :

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -1,3 +1,4 @@
    +const webpack = require('webpack');
     const Encore = require('@symfony/webpack-encore');
     const HtmlWebpackPlugin = require('html-webpack-plugin');

    @@ -10,6 +11,9 @@ Encore
         .enableSassLoader()
         .enableSingleRuntimeChunk()
         .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
    +    .addPlugin(new webpack.DefinePlugin({
    +        'ENV_API_ENDPOINT': JSON.stringify(process.env.API_ENDPOINT),
    +    }))
     ;

     module.exports = Encore.getWebpackConfig();

La variable d'environnement ``API_ENDPOINT`` doit pointer vers le serveur du site web où nous hébergeons le point d'entrée de l'API, ``/api``. Nous le configurerons correctement plus tard quand nous exécuterons ``yarn encore``.

Créez un fichier ``api.js`` qui abstrait la récupération des données de l'API :

.. code-block:: text
    :caption: src/api/api.js

    function fetchCollection(path) {
        return fetch(ENV_API_ENDPOINT + path).then(resp => resp.json()).then(json => json['hydra:member']);
    }

    export function findConferences() {
        return fetchCollection('api/conferences');
    }

    export function findComments(conference) {
        return fetchCollection('api/comments?conference='+conference.id);
    }

Vous pouvez maintenant adapter l'en-tête et les composants de l'accueil :

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -2,11 +2,23 @@ import '../assets/styles/app.scss';

     import {h, render} from 'preact';
     import {Router, Link} from 'preact-router';
    +import {useState, useEffect} from 'preact/hooks';

    +import {findConferences} from './api/api';
     import Home from './pages/home';
     import Conference from './pages/conference';

     function App() {
    +    const [conferences, setConferences] = useState(null);
    +
    +    useEffect(() => {
    +        findConferences().then((conferences) => setConferences(conferences));
    +    }, []);
    +
    +    if (conferences === null) {
    +        return <div className="text-center pt-5">Loading...</div>;
    +    }
    +
         return (
             <div>
                 <header className="header">
    @@ -19,15 +31,17 @@ function App() {
                     </nav>

                     <nav className="bg-light border-bottom text-center">
    -                    <Link className="nav-conference" href="/conference/amsterdam2019">
    -                        Amsterdam 2019
    -                    </Link>
    +                    {conferences.map((conference) => (
    +                        <Link className="nav-conference" href={'/conference/'+conference.slug}>
    +                            {conference.city} {conference.year}
    +                        </Link>
    +                    ))}
                     </nav>
                 </header>

                 <Router>
    -                <Home path="/" />
    -                <Conference path="/conference/:slug" />
    +                <Home path="/" conferences={conferences} />
    +                <Conference path="/conference/:slug" conferences={conferences} />
                 </Router>
             </div>
         )
    --- a/src/pages/home.js
    +++ b/src/pages/home.js
    @@ -1,7 +1,28 @@
     import {h} from 'preact';
    +import {Link} from 'preact-router';
    +
    +export default function Home({conferences}) {
    +    if (!conferences) {
    +        return <div className="p-3 text-center">No conferences yet</div>;
    +    }

    -export default function Home() {
         return (
    -        <div>Home</div>
    +        <div className="p-3">
    +            {conferences.map((conference)=> (
    +                <div className="card border shadow-sm lift mb-3">
    +                    <div className="card-body">
    +                        <div className="card-title">
    +                            <h4 className="font-weight-light">
    +                                {conference.city} {conference.year}
    +                            </h4>
    +                        </div>
    +
    +                        <Link className="btn btn-sm btn-blue stretched-link" href={'/conference/'+conference.slug}>
    +                            View
    +                        </Link>
    +                    </div>
    +                </div>
    +            ))}
    +        </div>
         );
    -};
    +}

Enfin, Preact Router passe le paramètre "slug" au composant Conference en tant que propriété. Utilisez-le pour afficher la conférence appropriée et ses commentaires, toujours en utilisant l'API ; et adaptez le rendu pour utiliser les données de l'API :

.. code-block:: diff
    :caption: patch_file

    --- a/src/pages/conference.js
    +++ b/src/pages/conference.js
    @@ -1,7 +1,48 @@
     import {h} from 'preact';
    +import {findComments} from '../api/api';
    +import {useState, useEffect} from 'preact/hooks';
    +
    +function Comment({comments}) {
    +    if (comments !== null && comments.length === 0) {
    +        return <div className="text-center pt-4">No comments yet</div>;
    +    }
    +
    +    if (!comments) {
    +        return <div className="text-center pt-4">Loading...</div>;
    +    }
    +
    +    return (
    +        <div className="pt-4">
    +            {comments.map(comment => (
    +                <div className="shadow border rounded-lg p-3 mb-4">
    +                    <div className="comment-img mr-3">
    +                        {!comment.photoFilename ? '' : (
    +                            <a href={ENV_API_ENDPOINT+'uploads/photos/'+comment.photoFilename} target="_blank">
    +                                <img src={ENV_API_ENDPOINT+'uploads/photos/'+comment.photoFilename} />
    +                            </a>
    +                        )}
    +                    </div>
    +
    +                    <h5 className="font-weight-light mt-3 mb-0">{comment.author}</h5>
    +                    <div className="comment-text">{comment.text}</div>
    +                </div>
    +            ))}
    +        </div>
    +    );
    +}
    +
    +export default function Conference({conferences, slug}) {
    +    const conference = conferences.find(conference => conference.slug === slug);
    +    const [comments, setComments] = useState(null);
    +
    +    useEffect(() => {
    +        findComments(conference).then(comments => setComments(comments));
    +    }, [slug]);

    -export default function Conference() {
         return (
    -        <div>Conference</div>
    +        <div className="p-3">
    +            <h4>{conference.city} {conference.year}</h4>
    +            <Comment comments={comments} />
    +        </div>
         );
    -};
    +}

La SPA a maintenant besoin de connaître l'URL de notre API grâce à la variable d'environnement ``API_ENDPOINT``. Définissez-la avec l'URL du serveur web de l'API (tournant dans le répertoire ``..``) :

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore dev

Vous pourriez aussi exécuter maintenant en arrière-plan :

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` symfony run -d --watch=webpack.config.js yarn encore dev --watch

Et l'application devrait maintenant fonctionner correctement dans le navigateur :

.. figure:: screenshots/spa-home-final.png
    :alt: /
    :align: center
    :figclass: with-browser spa

.. figure:: screenshots/spa-conference-final.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser spa

Wow ! Nous avons à présent une SPA entièrement fonctionnelle, avec routeur et données réelles. Nous pourrions organiser l'application Preact davantage si nous le voulions, mais elle fonctionne déjà très bien.

Déployer la SPA en production
-----------------------------

.. index::
    single: SymfonyCloud;Multi-Applications

SymfonyCloud permet de déployer plusieurs applications par projet. L'ajout d'une autre application peut se faire en créant un fichier ``.symfony.cloud.yaml`` dans n'importe quel sous-répertoire. Créez-en un sous ``spa/`` nommé ``spa`` :

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :emphasize-lines: 1

    name: spa

    type: php:8.0
    size: S

    build:
        flavor: none

    web:
        commands:
            start: sleep
        locations:
            "/":
                root: "public"
                index:
                    - "index.html"
                scripts: false
                expires: 10m

    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2
                unset NPM_CONFIG_PREFIX
                export NVM_DIR=${SYMFONY_APP_DIR}/.nvm

                yarn-install

                set +x && . "${SYMFONY_APP_DIR}/.nvm/nvm.sh" && set -x

                yarn encore prod
            )

.. index::
    single: SymfonyCloud;Routes

Modifiez le fichier ``.symfony/routes.yaml`` pour faire pointer le sous-domaine ``spa.`` vers l'application ``spa`` stockée dans le répertoire racine du projet :

.. code-block:: bash

    $ cd ../

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 4,5

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,5 @@
    +"https://spa.{all}/": { type: upstream, upstream: "spa:http" }
    +"http://spa.{all}/": { type: redirect, to: "https://spa.{all}/" }
    +
     "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Configurer CORS pour la SPA
---------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Si vous déployez le code maintenant, il ne fonctionnera pas car les navigateurs bloqueraient la requête à l'API. Nous devons explicitement autoriser la SPA à accéder à l'API. Récupérez le nom de domaine correspondant à votre application :

.. code-block:: bash

    $ symfony env:urls --first

Définissez la variable d'environnement ``CORS_ALLOW_ORIGIN`` en conséquence :

.. code-block:: bash

    $ symfony var:set "CORS_ALLOW_ORIGIN=^`symfony env:urls --first | sed 's#/$##' | sed 's#https://#https://spa.#'`$"

Si votre domaine est ``https://master-5szvwec-hzhac461b3a6o.eu.s5y.io/``, les appels ``sed`` le convertiront en ``https://spa.master-5szvwec-hzhac461b3a6o.eu.s5y.io``.

Nous devons également définir la variable d'environnement ``API_ENDPOINT`` :

.. code-block:: bash

    $ symfony var:set API_ENDPOINT=`symfony env:urls --first`

Commitez et déployez :

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -a -m'Add the SPA application'
    $ symfony deploy

Accédez à la SPA dans un navigateur en spécifiant l'application comme option :

.. code-block:: bash
    :class: ignore

    $ symfony open:remote --app=spa

Utiliser Cordova pour construire une application mobile
-------------------------------------------------------

.. index::
    single: SPA;Cordova
    single: Apache Cordova
    single: Cordova

**Apache Cordova** est un outil qui génère des applications mobiles multiplateformes. Et bonne nouvelle, il peut utiliser la SPA que nous venons de créer.

Installons-le :

.. code-block:: bash

    $ cd spa
    $ yarn global add cordova

.. note::

    Vous devez également installer le SDK Android. Cette section ne mentionne qu'Android, mais Cordova fonctionne avec toutes les plateformes mobiles, y compris iOS.

Créez la structure des répertoires de l'application :

.. code-block:: bash
    :class: answers(n)

    $ ~/.yarn/bin/cordova create app

Et générez l'application Android :

.. code-block:: bash
    :class: ignore

    $ cd app
    $ ~/.yarn/bin/cordova platform add android
    $ cd ..

C'est tout ce dont vous avez besoin. Vous pouvez maintenant *builder* les fichiers de production et les déplacer vers Cordova :

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore production
    $ rm -rf app/www
    $ mkdir -p app/www
    $ cp -R public/ app/www

Exécutez l'application sur un smartphone ou un émulateur :

.. code-block:: bash
    :class: ignore

    $ ~/.yarn/bin/cordova run android

.. sidebar:: Aller plus loin

    * `Le site officiel de Preact <https://preactjs.com/>`_ ;

    * `Le site officiel de Cordova <https://cordova.apache.org/>`_.

.. _`preact`: https://preactjs.com/
