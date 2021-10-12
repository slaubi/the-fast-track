Construirea unui SPA
====================

.. index::
    single: SPA
    single: Mobile

Majoritatea comentariilor vor fi transmise în timpul conferinței în care unii oameni nu aduc laptop. Dar probabil au un smartphone. Ce zici de crearea unei aplicații mobile pentru a verifica rapid comentariile conferinței?

Un mod de a crea o astfel de aplicație mobilă este de a construi o aplicație cu o singură pagină Javascript (SPA). Un SPA rulează local, poate utiliza spațiu de stocare local, poate apela la un API HTTP de pe un server și poate gestiona procese independente pentru a crea o experiență aproape nativă.

Crearea aplicației
-------------------

Pentru a crea aplicația mobilă, vom folosi `Preact`_ și **Symfony Encore**. **Preact** este o fundație mică și eficientă, potrivită pentru Guestbook SPA.

Pentru a face cât mai consistent atât site-ul, cât și SPA, vom reutiliza fișierele de stil Sass ale site-ului pentru aplicația mobilă.

Creează aplicația SPA în directorul ``spa`` și copie fișierele de stil ale site-ului:

.. code-block:: bash

    $ mkdir -p spa/src spa/public spa/assets/styles
    $ cp assets/styles/*.scss spa/assets/styles/
    $ cd spa

.. note::

    Am creat un director ``public``, deoarece vom interacționa în principal cu SPA prin intermediul unui browser. L-am fi putut numi ``build`` doar dacă am fi dorit să construim o aplicație mobilă.

Pentru o măsură bună, adaugă un fișier ``.gitignore``:

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

Acum, adaugă unele dependențe necesare:

.. code-block:: bash

    $ yarn add @symfony/webpack-encore @babel/core @babel/preset-env babel-preset-preact preact html-webpack-plugin bootstrap

Ultima etapă de configurare este crearea configurației Webpack Encore:

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

Crearea modelului principal SPA
-------------------------------

A sosit momentul pentru a crea șablonul inițial în care Preact va reda aplicația:

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

Elementul ``<div>`` este locul în care aplicația va fi redată prin JavaScript. Iată prima versiune a codului care redă „Hello World”:

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

Ultima linie înregistrează funcția ``App()`` pe elementul ``#app`` al paginii HTML.

Totul este gata acum!

Rularea unui SPA în browser
----------------------------

.. index::
    single: Symfony CLI;server:start
    single: Symfony CLI;server:stop

Deoarece această aplicație este independentă de site-ul principal, trebuie să rulăm un alt server web:

.. code-block:: bash
    :class: hide

    $ symfony server:stop

.. code-block:: bash

    $ symfony server:start -d --passthru=index.html

Opțiunea ``--passthru`` spune serverului web să treacă toate cererile HTTP la fișierul ``public/index.html`` (``public/`` este directorul rădăcină web implicit al serverului web). Această pagină este administrată de aplicația Preact și primește pagina să se redea prin istoricul „browserului”.

Pentru a compila fișierele CSS **și JavaScript**, rulează ``yarn``:

.. code-block:: bash

    $ yarn encore dev

Deschide SPA-ul într-un browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Și privește SPA-ul nostru de întâmpinare:

.. figure:: screenshots/spa.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Adăugarea unui router pentru gestionarea stărilor
---------------------------------------------------

În prezent, SPA-ul nu este capabi să gestioneze diferite pagini. Pentru a implementa mai multe pagini, avem nevoie de un router, precum cel pentru Symfony. Vom folosi **preact-router**. Preia o adresă URL ca intrare și o potrivește cu o componentă Preact care va fi afișat.

Instalează router-ul preact:

.. code-block:: bash

    $ yarn add preact-router

Creează o pagină inițială (o componentă *Preact*):

.. code-block:: text
    :caption: src/pages/home.js

    import {h} from 'preact';

    export default function Home() {
        return (
            <div>Home</div>
        );
    };

Și alta pentru pagina conferinței:

.. code-block:: text
    :caption: src/pages/conference.js

    import {h} from 'preact';

    export default function Conference() {
        return (
            <div>Conference</div>
        );
    };

Înlocuește „Hello World” ``div`` cu componenta ``Router``:

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

Reconstruiește aplicația:

.. code-block:: bash

    $ yarn encore dev

Dacă reîmprospătezi aplicația în browser, poți face acum clic pe linkurile „Home” și conferință. Reține că URL-ul browserului și butoanele înapoi/înainte ale browserului funcționează așa cum te aștepți.

Stilarea SPA
------------

În ceea ce privește site-ul, să adăugăm încărcătorul Sass:

.. code-block:: bash

    $ yarn add node-sass sass-loader

Activează încărcătorul Sass din Webpack și adaugă o referință la fișierul de stilzare:

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

Acum putem actualiza aplicația pentru a utiliza fișierele de stil:

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

Reconstruiește aplicația încă o dată:

.. code-block:: bash

    $ yarn encore dev

Acum te poți bucura de un SPA complet stilat:

.. figure:: screenshots/spa-home.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Preluarea datelor de la API
---------------------------

Structura aplicației Preact este acum finisată: Preact Router gestionează stările de pagină - inclusiv locul pentru identificatorul de cale - iar fișierul de stil principal este folosit pentru stilul SPA.

Pentru a face SPA-ul dinamic, trebuie să preluăm datele din API prin apeluri HTTP.

Configurează Webpack pentru a expune variabila de mediu endpoint API:

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

Variabila de mediu ``API_ENDPOINT`` ar trebui să indice către serverul web al site-ului web unde avem punctul final API sub ``/api``. O vom configura corect atunci când vom rula ``yarn encore`` în curând.

Creează un fișier ``api.js`` care abstractizează preluarea datelor din API:

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

Poți adapta acum antetul și componentele paginei inițiale:

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

În cele din urmă, Preact Router transmite „identificatorul de cale” componentei Conference ca proprietate. Folosește-l pentru a afișa conferința și comentariile sale, folosind din nou API-ul; și adaptează randarea să folosească datele din API:

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

SPA acum trebuie să cunoască adresa URL a API-ului nostru, prin intermediul variabilei de mediu ``API_ENDPOINT``. Seteaz-o la adresa URL a serverului web API (care rulează în directorul ``..``):

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore dev

De asemenea, poți rula în fundal acum:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` symfony run -d --watch=webpack.config.js yarn encore dev --watch

Iar aplicația din browser ar trebui să funcționeze corect:

.. figure:: screenshots/spa-home-final.png
    :alt: /
    :align: center
    :figclass: with-browser spa

.. figure:: screenshots/spa-conference-final.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser spa

Wow! Avem acum un SPA complet funcțional, cu router și date reale. Am putea organiza aplicația Preact în continuare dacă dorim, dar funcționează deja excelent.

Lansarea SPA-ului în producție
--------------------------------

.. index::
    single: SymfonyCloud;Multi-Applications

SymfonyCloud permite implementarea mai multor aplicații pentru fiecare proiect. Adăugarea unei alte aplicații se poate face prin crearea unui fișier ``.symfony.cloud.yaml`` în orice subdirector. Creează unul în ``spa/`` numit ``spa``:

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

Modifică fișierul ``.symfony/routes.yaml`` pentru a ruta subdomeniul ``spa.`` către aplicația ``spa`` stocată în directorul rădăcină al proiectului:

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

Configurarea CORS pentru SPA
----------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Dacă implementezi codul acum, acesta nu va funcționa deoarece un browser ar bloca solicitarea API. Trebuie să permitem în mod explicit SPA-ului să acceseze API-ul. Obține numele de domeniu curent atașat aplicației tale:

.. code-block:: bash

    $ symfony env:urls --first

Definește variabila de mediu ``CORS_ALLOW_ORIGIN`` în consecință:

.. code-block:: bash

    $ symfony var:set "CORS_ALLOW_ORIGIN=^`symfony env:urls --first | sed 's#/$##' | sed 's#https://#https://spa.#'`$"

Dacă domeniul tău este ``https://master-5szvwec-hzhac461b3a6o.eu.s5y.io/``, apelurile ``sed`` îl vor converti în ``https://spa.master-5szvwec-hzhac461b3a6o.eu.s5y.io``.

De asemenea, trebuie să setăm variabila de mediu ``API_ENDPOINT``:

.. code-block:: bash

    $ symfony var:set API_ENDPOINT=`symfony env:urls --first`

Salvează și lansează în producție:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -a -m'Add the SPA application'
    $ symfony deploy

Accesați SPA-ul într-un browser specificând aplicația ca steag:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote --app=spa

Folosind Cordova pentru a crea o aplicație pentru smartphone
-------------------------------------------------------------

.. index::
    single: SPA;Cordova
    single: Apache Cordova
    single: Cordova

**Apache Cordova** este un instrument care creează aplicații pentru smartphone-uri pe platforme multiple. Și o veste bună, poate folosi SPA-ul pe care tocmai l-am creat.

Să-l instalăm:

.. code-block:: bash

    $ cd spa
    $ yarn global add cordova

.. note::

    De asemenea, trebuie să instalezi SDK-ul pentru Android. Această secțiune menționează doar Android, dar Cordova funcționează cu toate platformele mobile, inclusiv cu iOS.

Creează structura directorului aplicației:

.. code-block:: bash
    :class: answers(n)

    $ ~/.yarn/bin/cordova create app

Și generează aplicația Android:

.. code-block:: bash
    :class: ignore

    $ cd app
    $ ~/.yarn/bin/cordova platform add android
    $ cd ..

Asta este tot ce îți trebuie. Poți acum să compilezi fișierele de producție și să le muți în Cordova:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore production
    $ rm -rf app/www
    $ mkdir -p app/www
    $ cp -R public/ app/www

Rulează aplicația pe un smartphone sau un emulator:

.. code-block:: bash
    :class: ignore

    $ ~/.yarn/bin/cordova run android

.. sidebar:: Mergând mai departe

    * `Site-ul oficial Preact <https://preactjs.com/>`_;

    * `Site-ul oficial Cordova <https://cordova.apache.org/>`_.

.. _`preact`: https://preactjs.com/
