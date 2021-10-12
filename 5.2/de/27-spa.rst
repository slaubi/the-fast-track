Aufbau einer SPA
================

.. index::
    single: SPA
    single: Mobile

Die meisten Kommentare werden während der Konferenz eingereicht, wo einige Leute keinen Laptop mitbringen. Wahrscheinlich haben sie jedoch ein Smartphone. Wie wäre es mit der Erstellung einer mobilen App, um schnell die Kommentare zur Konferenz lesen zu können?

Eine Möglichkeit, eine solche mobile Anwendung zu erstellen, ist die Erstellung einer Javascript Single Page Application (SPA). Eine SPA läuft lokal, kann den Local-Storage verwenden, kann eine Remote-HTTP-API aufrufen und Service-Worker nutzen, um eine nahezu native Erfahrung zu schaffen.

Die Anwendung erstellen
-----------------------

Um die mobile Anwendung zu erstellen, werden wir `Preact`_ und **Symfony Encore** verwenden. **Preact** ist eine kleine und effiziente Basis, die sich gut für die Gästebuch SPA eignet.

Um sowohl die Website als auch die SPA konsistent zu machen, werden wir die Sass-Stylesheets der Website für die mobile Anwendung wiederverwenden.

Erstelle die SPA-Anwendung unterhalb des ``spa``-Verzeichnisses und kopiere die Stylesheets der Website:

.. code-block:: bash

    $ mkdir -p spa/src spa/public spa/assets/styles
    $ cp assets/styles/*.scss spa/assets/styles/
    $ cd spa

.. note::

    Wir haben ein ``public``-Verzeichnis erstellt, da wir hauptsächlich über einen Browser mit der SPA interagieren werden. Wir hätten es ``build`` nennen können, wenn wir lediglich eine mobile Anwendung entwickeln wollten.

Füge sicherheitshalber eine ``.gitignore``-Datei hinzu:

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

Füge nun einige erforderliche Dependencies hinzu:

.. code-block:: bash

    $ yarn add @symfony/webpack-encore @babel/core @babel/preset-env babel-preset-preact preact html-webpack-plugin bootstrap

Der letzte Konfigurationsschritt besteht darin, die Webpack Encore-Konfiguration zu erstellen:

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

Das SPA Haupt-Template erstellen
--------------------------------

Zeit, das initiale Template zu erstellen, in der Preact die Anwendung rendern wird:

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

Der ``<div>``-Tag ist der Ort, an dem die Anwendung per JavaScript dargestellt wird. Hier ist die erste Version des Codes, der die "Hello World"-Ansicht darstellt:

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

Die letzte Zeile registriert die ``App()``-Funktion auf dem ``#app``-Element der HTML-Seite.

Jetzt ist alles bereit!

Eine SPA im Browser ausführen
-----------------------------

.. index::
    single: Symfony CLI;server:start
    single: Symfony CLI;server:stop

Da diese Anwendung unabhängig von der Haupt-Website ist, müssen wir einen anderen Webserver betreiben:

.. code-block:: bash
    :class: hide

    $ symfony server:stop

.. code-block:: bash

    $ symfony server:start -d --passthru=index.html

Das ``--passthru``-Flag weist den Webserver an, alle HTTP-Requests an die ``public/index.html``-Datei zu übergeben (``public/`` ist das Standard-Web-Root-Verzeichnis des Webservers). Diese Seite wird von der Preact-Anwendung verwaltet und ermittelt die zu rendernde Seite über den Pfad im Browser.

Um die CSS **und die JavaScript**-Dateien zu kompilieren, führe ``yarn`` aus:

.. code-block:: bash

    $ yarn encore dev

Öffne die SPA in einem Browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Und schau Dir unsere Hallo-Welt SPA an:

.. figure:: screenshots/spa.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Einen Router zur Behandlung von Zuständen hinzufügen
----------------------------------------------------

Die SPA ist derzeit nicht in der Lage, verschiedene Seiten zu verarbeiten. Um mehrere Seiten zu implementieren, benötigen wir einen Router, wie bei Symfony. Wir werden den **preact-router** verwenden. Er nimmt eine URL als Input und ordnet sie einer Preact-Komponente zu, die angezeigt werden soll.

Installiere den Preact-Router:

.. code-block:: bash

    $ yarn add preact-router

Erstelle eine Seite für die Homepage (eine *Preact-Komponente*):

.. code-block:: text
    :caption: src/pages/home.js

    import {h} from 'preact';

    export default function Home() {
        return (
            <div>Home</div>
        );
    };

Und noch eine für die Konferenzseite:

.. code-block:: text
    :caption: src/pages/conference.js

    import {h} from 'preact';

    export default function Conference() {
        return (
            <div>Conference</div>
        );
    };

Ersetze das "Hello World"-``div`` mit der ``Router``-Komponente:

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

Erstelle die Anwendung neu:

.. code-block:: bash

    $ yarn encore dev

Wenn Du die Anwendung im Browser aktualisierst, kannst Du nun auf die Links "Home" und "Konferenz" klicken. Du siehst, dass die Browser-URL und die Vor- und Zurück-Buttons Deines Browsers so funktionieren, wie Du es erwarten würdest.

Die SPA gestalten
-----------------

Lass uns den Sass-Loader zur Website hinzufügen:

.. code-block:: bash

    $ yarn add node-sass sass-loader

Aktiviere den Sass-Loader in Webpack und füge eine Referenz auf das Stylesheet hinzu:

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

Wir können nun die Anwendung aktualisieren, um die Stylesheets zu verwenden:

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

Erstelle die Anwendung noch einmal neu:

.. code-block:: bash

    $ yarn encore dev

Du kommst nun in den Genuss einer komplett gestylten SPA:

.. figure:: screenshots/spa-home.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Daten aus der API holen
-----------------------

Die Preact-Anwendungsstruktur ist nun fertig: Der Preact-Router verarbeitet die Seitenzustände – einschließlich des Platzhalters für den Konferenz-Slug – und das Stylesheet der Hauptanwendung wird zur Gestaltung des SPA verwendet.

Um die SPA dynamisch zu machen, müssen wir die Daten mittels HTTP-Requests aus der API holen.

Konfiguriere Webpack, um die Environment-Variable für die API-Endpunkte zu definieren:

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

Die Environment-Variable ``API_ENDPOINT`` sollte auf den Webserver der Website zeigen, wo wir unter ``/api`` den API-Endpunkt haben. Wir werden sie bald ordnungsgemäß konfigurieren, wenn wir ``yarn encore`` ausführen.

Erstelle eine ``api.js``-Datei, die den Datenabruf aus der API abstrahiert:

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

Du kannst nun die Header- und Home-Komponenten anpassen:

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

Schließlich übergibt der Preact Router den Platzhalter "slug" als Eigenschaft an die Konferenz-Komponente. Verwende ihn um die richtige Konferenz und ihre Kommentare darzustellen, wobei du wieder die API nutzt; passe außerdem das Rendering an, um die API-Daten zu verwenden:

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

Die SPA muss nun die URL zu unserer API über die Environment-Variable ``API_ENDPOINT`` kennen. Setze sie auf die API-Webserver-URL (die im ``..``-Verzeichnis läuft):

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore dev

Du könntest es jetzt auch im Hintergrund laufen lassen:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` symfony run -d --watch=webpack.config.js yarn encore dev --watch

Die Anwendung sollte nun im Browser einwandfrei funktionieren:

.. figure:: screenshots/spa-home-final.png
    :alt: /
    :align: center
    :figclass: with-browser spa

.. figure:: screenshots/spa-conference-final.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser spa

Wow! Wir haben jetzt eine voll funktionsfähige SPA mit Router und echten Daten. Wir könnten die Preact-App weiter organisieren, wenn wir wollen, aber sie funktioniert bereits hervorragend.

Die SPA zum Produktivsystem deployen
------------------------------------

.. index::
    single: SymfonyCloud;Multi-Applications

SymfonyCloud ermöglicht es, mehrere Anwendungen pro Projekt zu deployen. Das Hinzufügen einer weiteren Anwendung kann durch Erstellen einer ``.symfony.cloud.yaml``-Datei in einem beliebigen Unterverzeichnis erfolgen. Erstelle eine unter ``spa/`` namens  ``spa``:

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

Bearbeite die ``.symfony/routes.yaml``-Datei, um die ``spa.``-Subdomain an die im Projekt-Stammverzeichnis gespeicherte ``spa``-Anwendung weiterzuleiten:

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

CORS für die SPA konfigurieren
------------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Wenn Du den Code jetzt deployst, funktioniert er nicht, da ein Browser die API-Anfrage blockieren würde. Wir müssen der SPA explizit den Zugriff auf die API erlauben. Hole Dir den aktuellen Domainnamen, der mit Deiner Anwendung verknüpft ist:

.. code-block:: bash

    $ symfony env:urls --first

Definiere die Environment-Variable ``CORS_ALLOW_ORIGIN`` entsprechend:

.. code-block:: bash

    $ symfony var:set "CORS_ALLOW_ORIGIN=^`symfony env:urls --first | sed 's#/$##' | sed 's#https://#https://spa.#'`$"

Wenn Deine Domain ``https://master-5szvwec-hzhac461b3a6o.eu.s5y.io/`` ist, wird sie durch die``sed``-Aufrufe zu  ``https://spa.master-5szvwec-hzhac461b3a6o.eu.s5y.io`` umgewandelt.

Wir müssen auch die Environment-Variable ``API_ENDPOINT`` setzen:

.. code-block:: bash

    $ symfony var:set API_ENDPOINT=`symfony env:urls --first`

Committe und Deploye:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -a -m'Add the SPA application'
    $ symfony deploy

Greife in einem Browser auf die SPA zu, indem Du die Anwendung als Flag angibst:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote --app=spa

Eine Smartphone-Anwendung mit Cordova erstellen
-----------------------------------------------

.. index::
    single: SPA;Cordova
    single: Apache Cordova
    single: Cordova

**Apache Cordova** ist ein Tool, das plattformübergreifende Smartphone-Anwendungen erstellt. Eine gute Nachricht, es kann die SPA nutzen, die wir gerade erstellt haben.

Lass es uns installieren:

.. code-block:: bash

    $ cd spa
    $ yarn global add cordova

.. note::

    Du musst auch das Android SDK installieren. Dieser Abschnitt erwähnt nur Android, aber Cordova funktioniert mit allen mobilen Plattformen, einschließlich iOS.

Erstelle die Verzeichnisstruktur der Anwendung:

.. code-block:: bash
    :class: answers(n)

    $ ~/.yarn/bin/cordova create app

Und generiere die Android-Applikation:

.. code-block:: bash
    :class: ignore

    $ cd app
    $ ~/.yarn/bin/cordova platform add android
    $ cd ..

Das ist alles, was Du brauchst. Du kannst nun die Produktivdateien erstellen und zu Cordova verschieben:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore production
    $ rm -rf app/www
    $ mkdir -p app/www
    $ cp -R public/ app/www

Führe die Anwendung auf einem Smartphone oder einem Emulator aus:

.. code-block:: bash
    :class: ignore

    $ ~/.yarn/bin/cordova run android

.. sidebar:: Weiterführendes

    * `Die offizielle Preact Website <https://preactjs.com/>`_;

    * `Die offizielle Cordova-Website <https://cordova.apache.org/>`_.

.. _`preact`: https://preactjs.com/
