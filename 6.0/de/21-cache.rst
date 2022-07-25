Performance durch Caching
=========================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Mit wachsender Popularität können Performance-Probleme einhergehen. Einige typische Beispiele: fehlende Datenbankindizes oder tonnenweise SQL-Abfragen pro Seite. Mit einer leeren Datenbank wirst du wirst keine Probleme haben, aber bei mehr Traffic und wachsenden Datenmengen könnten irgendwann Performance-Probleme auftreten.

HTTP-Cache-Header hinzufügen
-----------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Die Verwendung von HTTP-Caching-Strategien ist eine großartige Möglichkeit, die Leistung für Endbenutzer*innen mit geringem Aufwand zu maximieren. Füge im Produktivsystem einen Reverse-Proxy-Cache hinzu, um das Caching zu ermöglichen, und verwende ein `CDN`_, um eine noch bessere Leistung zu erzielen.

Lass uns die Homepage für eine Stunde cachen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -33,9 +33,12 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/index.html.twig', [
    +        $response = new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         #[Route('/conference/{slug}', name: 'conference')]

Die ``setSharedMaxAge()``-Methode konfiguriert die Cache-Dauer für Reverse-Proxies. Verwende ``setMaxAge()``, um den Browser-Cache zu steuern. Die Zeit wird in Sekunden angegeben (1 Stunde = 60 Minuten = 3600 Sekunden).

Das Cachen der Konferenzseite ist schwieriger, da sie dynamischer ist. Jede*r kann jederzeit einen Kommentar hinzufügen, und niemand will eine Stunde warten, um ihn online zu sehen. Verwende in solchen Fällen die *HTTP-Validation*-Strategie.

Den Symfony HTTP Cache Kernel aktivieren
----------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Aktiviere den Symfony HTTP-Reverse-Proxy um die HTTP-Cache-Strategie zu testen, aber nur in der "development"-Environment (Für die "production" Environment werden wir eine robustere Lösung nutzen):

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Der Symfony HTTP-Reverse-Proxy (über die ``HttpCache``-Klasse) ist nicht nur ein vollwertiger HTTP-Reverse-Proxy, sondern fügt auch einige nützliche Debug-Informationen als HTTP-Header hinzu. Das hilft sehr bei der Validierung der von uns eingestellten Cache-Header.

Überprüfe es auf der Homepage:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

Für den allerersten Request teilt Dir der Cache-Server mit, dass es sich um einen ``miss`` handelt und dass er ``store`` ausführte, um die Response zu cachen. Überprüfe den ``cache-control``-Header, um die konfigurierte Cache Strategie zu sehen.

Für nachfolgende Requests ist die Response im Cache (``age`` wurde ebenfalls aktualisiert):

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

SQL-Requests mit ESI vermeiden
------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Der ``TwigEventSubscriber``-Listener injiziert eine globale Variable mit allen Konferenzobjekten in Twig. Dies geschieht für jede einzelne Seite der Webseite. Das ist wahrscheinlich ein großer Optimierungspunkt.

Du wirst nicht jeden Tag neue Konferenzen hinzufügen, sodass der Code immer wieder genau die gleichen Daten aus der Datenbank abfragt.

Wir möchten vielleicht die Konferenznamen und Slugs mit dem Symfony Cache zwischenspeichern. Ich verlasse mich jedoch, wann immer es möglich ist, auf die HTTP-Caching-Infrastruktur.

Wenn Du ein Fragment einer Seite zwischenspeichern möchtest, verschiebe es aus dem aktuellen HTTP-Requests, indem Du einen *Sub-Request* erstellst. *ESI* ist die perfekte Ergänzung zu diesem Anwendungsfall. Ein ESI ist eine Möglichkeit, das Ergebnis einer HTTP-Anfrage in eine andere einzubetten.

Erstelle einen Controller, der nur das HTML-Fragment zurückgibt, welches die Konferenzen anzeigt:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -41,6 +41,14 @@ class ConferenceController extends AbstractController
             return $response;
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return new Response($this->twig->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
    +    }
    +
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {

Erstelle das entsprechende Template:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Rufe ``/conference_header`` im Browser auf, um zu überprüfen, ob alles in Ordnung ist.

.. index::
    single: Twig;render
    single: Twig;path

Zeit, den Trick zu enthüllen! Aktualisiere das Twig-Template, um den soeben erstellten Controller aufzurufen:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,11 +16,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Und *voilà*. Aktualisiere die Seite und sie zeigt immer noch das Gleiche an.

.. tip::

    Verwende das Symfony Profiler-Panel "Request / Response", um mehr über den Main Request und seine Sub-Requests zu erfahren.

Nun werden bei jedem Aufruf einer Seite im Browser zwei HTTP-Requests ausgeführt, einer für den Header und einer für die Hauptseite. Du hast die Performance verschlechtert. Herzlichen Glückwunsch!

Der Konferenz Header HTTP-Request wird derzeit intern von Symfony durchgeführt, so dass kein HTTP-Round-trip erforderlich ist. Dies bedeutet auch, dass es keine Möglichkeit gibt, von HTTP-Cache-Headern zu profitieren.

Konvertiere den Request in einen "echten" HTTP-Request mit Hilfe von ESI.

Aktiviere zunächst den ESI-Support:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_samesite: lax
             storage_factory_id: session.storage.factory.native

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Verwende ``render_esi`` anstelle von ``render``:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,7 +16,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Wenn Symfony einen Reverse-Proxy erkennt, der mit ESIs umgehen kann, aktiviert es den ESI-Support automatisch (wenn nicht, greift es auf ``render`` zurück, um den Sub-Request synchron auszuführen).

Da der Symfony Reverse-Proxy ESIs unterstützt, sollten wir seine Logs überprüfen (zuerst den Cache entfernen – siehe "Purging" unten):

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

Aktualisiere die Seite einige Male: Die ``/``-Response wird zwischengespeichert und die ``/conference_header``-Response nicht. Wir haben etwas Großartiges erreicht: Wir haben die ganze Seite im Cache, aber ein Teil ist immer noch dynamisch.

Das ist aber nicht das, was wir wollen. Cache die Header-Seite für eine Stunde, unabhängig von allem anderen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -44,9 +44,12 @@ class ConferenceController extends AbstractController
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/header.html.twig', [
    +        $response = new Response($this->twig->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         #[Route('/conference/{slug}', name: 'conference')]

Der Cache ist nun für beide Requests aktiviert:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

Der ``x-symfony-cache``-Header enthält zwei Elemente: den Main Request ``/`` und einen Sub-Request (den ``conference_header`` ESI). Beide befinden sich im Cache (``fresh``).

Die Cache-Strategie kann sich von der Hauptseite und ihren ESIs unterscheiden. Wenn wir eine "Über"-Seite haben, möchten wir sie vielleicht für eine Woche im Cache speichern und trotzdem den Header jede Stunde aktualisieren lassen.

Entferne den Listener, da wir ihn nicht mehr benötigen:

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Den HTTP-Cache zum Testen leeren
--------------------------------

Das Testen der Website in einem Browser oder durch automatisierte Tests wird mit einer Caching-Ebene etwas schwieriger.

Du kannst den gesamten HTTP-Cache manuell entfernen, indem Du das ``var/cache/dev/http_cache/``-Verzeichnis entfernst:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

Diese Strategie funktioniert nicht gut, wenn Du nur einige URLs invalidieren willst oder wenn Du die Cache-Invalidierung in Deine Funktionalen Tests integrieren willst. Lass uns einen kleinen HTTP-Endpunkt nur für Administrator*innen hinzufügen, um einige URLs zu invalidieren:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -36,3 +36,5 @@ services:
             tags:
                 - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
                 - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}
    +
    +    Symfony\Component\HttpKernel\HttpCache\StoreInterface: '@http_cache.store'
    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
    @@ -52,4 +54,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

Der neue Controller wurde auf die ``PURGE``-HTTP-Methode beschränkt. Diese Methode ist nicht im HTTP-Standard enthalten, wird aber häufig verwendet, um Caches zu invalidieren.

Standardmäßig können Routenparameter kein ``/``-Zeichen enthalten, da es URL-Segmente trennt. Du kannst diese Einschränkung für den letzten Routenparameter, hier beispielsweise ``uri``, überschreiben, indem Du für ihn ein eigenes Requirement-Pattern (``.*``) festlegst.

Die Art und Weise, wie wir die ``HttpCache``-Instanz erhalten, mag etwas seltsam wirken; wir verwenden eine anonyme Klasse, da der Zugriff auf die "echte" Klasse nicht möglich ist. Die ``HttpCache``-Instanz umschließt den echten Kernel, welcher den Cache-Layer nicht kennt, was genau so sein sollte.

Invalidiere die Homepage und den Konferenz-Header mittels folgender cURL-Aufrufe:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

Der ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL``-Unterbefehl gibt die aktuelle URL des lokalen Webservers zurück.

.. note::

    Der Controller hat keinen Routennamen, da er im Code nie referenziert werden wird.

Ähnliche Routen mit einem Präfix gruppieren
---------------------------------------------

.. index::
    single: Attributes;Route

Die beiden Routen im Admin-Controller haben das gleiche ``/admin``-Präfix. Anstatt es in allen Routen zu wiederholen, überarbeiten wir die Routen und konfigurieren das Präfix an der Klasse selbst:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         private $twig;
    @@ -28,7 +29,7 @@ class AdminController extends AbstractController
             $this->bus = $bus;
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -55,7 +56,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

CPU/Speicher-intensive Operationen cachen
-----------------------------------------

.. index::
    single: Process
    single: Components;Process

Wir haben keine CPU- oder speicherintensiven Algorithmen auf der Website. Um über *lokale Caches* zu sprechen, erstellen wir einen Befehl, der den aktuellen Schritt anzeigt, an dem wir gerade arbeiten (genauer gesagt, den Git-Tag-Namen, der an den aktuellen Git-Commit angehängt ist).

Die Symfony Process-Komponente ermöglicht es Dir, einen Befehl auszuführen und das Ergebnis zurückzubekommen (Standard- und Fehlerausgabe).

Implementiere den Befehl:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    class StepInfoCommand extends Command
    {
        protected static $defaultName = 'app:step:info';

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return 0;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Du hättest ``make:command`` nutzen können, um den Befehl zu erstellen:

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Was, wenn wir die Ausgabe für ein paar Minuten cachen wollen? Verwende den Symfony Cache.

Und umschließe den Code mit der Cache-Logik:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
    @@ -6,16 +6,31 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     class StepInfoCommand extends Command
     {
         protected static $defaultName = 'app:step:info';

    +    private $cache;
    +
    +    public function __construct(CacheInterface $cache)
    +    {
    +        $this->cache = $cache;
    +
    +        parent::__construct();
    +    }
    +
         protected function execute(InputInterface $input, OutputInterface $output): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $this->cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return 0;
         }

Der Prozess wird nun nur noch aufgerufen, wenn sich das ``app.current_step``-Element nicht im Cache befindet.

Profilerstellung und Performance-Vergleiche
-------------------------------------------

Füge niemals blindlings Caching hinzu. Behalte im Hinterkopf, dass das Hinzufügen eines Caches mehr Komplexität bedeutet. Und da wir alle sehr schlecht darin sind, zu erraten, was schnell und was langsam ist, könntest Du in eine Situation geraten, in welcher der Cache Deine Anwendung langsamer macht.

Messe immer die Auswirkungen des Hinzufügens eines Cache mit einem Profiler-Tool wie `Blackfire`_.

Gehe zum Schritt über "Performance", um mehr darüber zu erfahren, wie Du Blackfire verwenden kannst, um Deinen Code vor dem Deployment zu testen.

Einen Reverse Proxy-Cache auf dem Produktivsystem konfigurieren
---------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Platform.sh;Varnish
    single: Varnish

Anstelle des Symfony-Reverse-Proxy, nutzen wir den mehr "robusten" Varnish-Reverse-Proxy auf dem Produktivsystem.

Füge Varnish zu den Platform.sh-Diensten hinzu:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -2,3 +2,12 @@
     database:
         type: postgresql:13
         disk: 1024
    +
    +varnish:
    +    type: varnish:6.0
    +    relationships:
    +        application: 'app:http'
    +    configuration:
    +        vcl: !include
    +            type: string
    +            path: config.vcl

.. index::
    single: Platform.sh;Routes

Verwende Varnish als Main-Entry-Point in den Routen:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/routes.yaml
    +++ b/.platform/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Erstelle zum Abschluss eine ``config.vcl``-Datei zur Konfiguration von Varnish:

.. code-block:: vcl
    :caption: .platform/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

ESI-Unterstützung für Varnish aktivieren
------------------------------------------

Die ESI-Unterstützung für Varnish sollte für jeden Request explizit aktiviert werden. Um es universell zu machen verwendet Symfony die standardisierten ``Surrogate-Capability``- und ``Surrogate-Control``-Header, um die ESI-Unterstützung auszuhandeln:

.. code-block:: vcl
    :caption: .platform/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Varnish Cache leeren
--------------------

Die Invalidierung des Caches auf dem Produktivsystem sollte wahrscheinlich nie erforderlich sein, außer für Notfallzwecke und vielleicht auf einem Nicht-``master``-Branch. Wenn Du den Cache häufig leeren musst, bedeutet dies wahrscheinlich, dass die Caching-Strategie optimiert werden sollte (durch Senkung der TTL oder durch Verwendung einer Validierungsstrategie anstelle einer Ablaufstrategie).

Wie auch immer, lasse uns sehen, wie man Varnish für die Invalidierung des Caches konfiguriert:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/config.vcl
    +++ b/.platform/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

Im wirklichen Leben würdest du das wahrscheinlich auf bestimmte IPs einschränken, wie in der `Varnish Dokumentation`_ beschrieben.

Purge jetzt einige URLs:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

Die URLs sehen etwas seltsam aus, da die von ``env:url`` zurückgegebenen URLs bereits mit ``/`` enden.

.. sidebar:: Weiterführendes

    * `Cloudflare`_, die globale Cloud-Plattform;

    * `Varnish HTTP Cache Dokumentation`_;

    * `ESI-Spezifikation`_ und `ESI-Entwicklerressourcen`_;

    * `HTTP-Cache Validierungsmodell`_;

    * `HTTP-Cache auf Platform.sh`_.

.. _`Blackfire`: https://blackfire.io/
.. _`Varnish Dokumentation`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Varnish HTTP Cache Dokumentation`: https://varnish-cache.org/docs/index.html
.. _`ESI-Spezifikation`: https://www.w3.org/TR/esi-lang
.. _`ESI-Entwicklerressourcen`: https://www.akamai.com/us/en/support/esi.jsp
.. _`HTTP-Cache Validierungsmodell`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP-Cache auf Platform.sh`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
