Caching voor performance
========================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Populariteit kan met performanceproblemen gepaard gaan. Enkele typische voorbeelden: ontbrekende database-indexen of een gigantische hoeveelheid aan SQL-verzoeken per pagina. Met een lege database zal je geen problemen ondervinden, maar met meer verkeer en steeds meer gegevens kan je op een gegeven moment in de problemen komen.

HTTP-Cache headers toevoegen
----------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Het gebruik van HTTP-cachingstrategieën is een geweldige manier om met weinig inspanning de prestaties voor eindgebruikers te maximaliseren. Voeg een reverse proxy-cache toe in productie om caching mogelijk te maken en gebruik een `CDN`_ om zo dicht mogelijk bij de eindgebruikers te cachen voor nog betere prestaties.

Laten we de homepage een uur cachen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -35,9 +35,12 @@ class ConferenceController extends AbstractController
          */
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

         /**

De methode ``setSharedMaxAge()`` configureert de vervaldatum van de cache voor reverse proxies. Gebruik ``setMaxAge()`` om de browsercache te beheren. De tijd wordt uitgedrukt in seconden (1 uur = 60 minuten = 3600 seconden).

Het cachen van de conferentiepagina is moeilijker, omdat deze dynamischer is. Iedereen kan op elk moment een reactie toevoegen en niemand wil een uur wachten om die online te zien. Gebruik in dergelijke gevallen de *HTTP-validatiestrategie*.

Het activeren van Symfony's HTTP-Cache kernel
---------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Om de HTTP-cachestrategie te testen, gebruik je Symfony's HTTP-reverse proxy:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -1,6 +1,7 @@
     <?php

     use App\Kernel;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;

    @@ -21,6 +22,11 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? false) {
     }

     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
    +
    +if ('dev' === $kernel->getEnvironment()) {
    +    $kernel = new HttpCache($kernel);
    +}
    +
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);

Naast het feit dat het een volwaardige HTTP-reverse proxy is, voegt Symfony's HTTP-reverse proxy (via de ``HttpCache`` class) wat fijne debuginfo toe, zoals HTTP-headers. Dat helpt enorm bij het valideren van de cache-headers die we hebben ingesteld.

Bekijk het op de homepage:

.. code-block:: bash
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

Bij de allereerste request vertelt de cacheserver je dat het een ``miss`` was en het een ``store`` heeft uitgevoerd om de response te cachen. Controleer de ``cache-control``-header om de geconfigureerde cachingstrategie te zien.

Voor daaropvolgende requests is de response gecachet (de ``age`` is ook bijgewerkt):

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

SQL-requests vermijden met ESI
------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

De ``TwigEventSubscriber``-listener injecteert een globale variabele in Twig met alle conferentieobjecten en doet dat voor elke pagina op de website. Daar valt vast een hoop aan te optimaliseren.

Je zal niet elke dag conferenties toevoegen, dus de code vraagt telkens exact dezelfde gegevens op uit de database.

We zouden de namen en slugs van de conferenties kunnen cachen met Symfony Cache, maar waar mogelijk wil ik graag vertrouwen op de HTTP-cachinginfrastructuur.

Wanneer je een deel van een pagina wilt cachen, koppel je die los van de huidige HTTP-request door een *subrequest* te maken. *ESI* is de beste oplossing voor deze use case. ESI maakt het mogelijk om het resultaat van een HTTP-request te integreren in een andere request.

Maak een controller aan die alleen het deel van de HTML terugstuurt dat de conferenties weergeeft:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -45,6 +45,16 @@ class ConferenceController extends AbstractController
             return $response;
         }

    +    /**
    +     * @Route("/conference_header", name="conference_header")
    +     */
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return new Response($this->twig->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
    +    }
    +
         /**
          * @Route("/conference/{slug}", name="conference")
          */

Maak de bijbehorende template aan:

.. code-block:: twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Check ``/conference_header`` om te kijken of alles goed werkt.

.. index::
    single: Twig;render
    single: Twig;path

Tijd om de truc te onthullen! Update de Twig-layout om de zojuist gemaakte controller op te roepen:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,11 +8,7 @@
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

En *voilà*. Vernieuw de pagina en de website wordt nog steeds hetzelfde weergegeven.

.. tip::

    Gebruik het "Request / Response" Symfony-profilerpaneel om meer te weten te komen over de hoofdrequest en de subrequests.

Elke keer dat je nu een pagina bezoekt in de browser worden er twee HTTP-requests uitgevoerd: één voor de header en één voor de main-pagina. Je hebt de performance slechter gemaakt. Gefeliciteerd!

De HTTP-call voor de header van de conferentiepagina wordt momenteel intern gedaan door Symfony, dus er is geen HTTP-round-trip aan te pas gekomen. Dit betekent ook dat er geen enkele manier is om te profiteren van HTTP-cache-headers.

Converteer de call naar een "echte" HTTP-call met behulp van een ESI.

Schakel eerst de ESI-ondersteuning in:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -11,7 +11,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Gebruik vervolgens ``render_esi`` in plaats van ``render``:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,7 +8,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Als Symfony een reverse proxy detecteert die weet hoe om te gaan met ESI's, wordt ESI automatisch ondersteund (zo niet, dan valt het systeem terug op het synchroon uitvoeren van de subrequest).

Laten we de logs bekijken, aangezien de reverse proxy van Symfony ESI's ondersteunt (verwijder eerst de cache - zie "Legen" hieronder):

.. code-block:: bash
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

Vernieuw de pagina een paar keer: de ``/`` response wordt gecachet en de ``/conference_header`` response niet. We hebben iets geweldigs bereikt: de hele pagina in de cache, maar één deel is nog steeds dynamisch.

Dit is echter niet wat we willen. Cache de headerpagina een uur lang, onafhankelijk van de rest:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -48,9 +48,12 @@ class ConferenceController extends AbstractController
          */
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

         /**

Cache is nu ingeschakeld voor beide requests:

.. code-block:: bash
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

De header ``x-symfony-cache`` bevat twee elementen: de hoofdrequest ``/`` en een subrequest (de ``conference_header`` ESI). Beide zitten in de cache (``fresh``).

De cachingstrategie kan verschillen van de hoofdpagina en diens ESI's. Als we een "about"-pagina hebben willen we die misschien een week cachen, en toch de header elk uur laten bijwerken.

Verwijder de listener, aangezien we die niet meer nodig hebben:

.. code-block:: bash

    $ rm src/EventSubscriber/TwigEventSubscriber.php

De HTTP-cache legen om te testen
--------------------------------

Het testen van de website in een browser of via geautomatiseerde tests wordt met een cachinglaag iets moeilijker.

You can manually remove all the HTTP cache by removing the
``var/cache/dev/http_cache/`` directory:

.. code-block:: bash

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Annotations;@Route

Deze strategie werkt niet goed als je alleen een paar URLs wilt invalideren of als je de cache wil invalideren voor je functionele tests. Laten we een klein, admin-only HTTP-endpoint toevoegen om wat URLs te invalideren:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -6,8 +6,10 @@ use App\Entity\Comment;
     use App\Message\CommentMessage;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
    @@ -54,4 +56,19 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]);
         }
    +
    +    /**
    +     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     */
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store = (new class($kernel) extends HttpCache {})->getStore();
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

De nieuwe controller is beperkt tot de ``PURGE`` HTTP-methode. Deze methode staat niet in de HTTP-standaard, maar wordt wel veel gebruikt om caches te invalideren.

Standaard kunnen routeparameters geen ``/`` bevatten, omdat dit de URL-segmenten scheidt. Je kan deze beperking voor de laatste routeparameter, bijvoorbeeld ``uri``, opheffen door jouw eigen patroon in te stellen waar de URL aan moet voldoen (``.*``).

De manier waarop we de ``HttpCache``-instantie krijgen kan er ook een beetje vreemd uitzien; we gebruiken een anonieme class omdat toegang tot de "echte" class niet mogelijk is. De ``HttpCache``-instantie omsluit de echte kernel, die zich, zoals het hoort, niet bewust is van de cachelaag.

Invalideer de homepage en de kop van de conferentiepagina via de volgende cURL-calls:

.. code-block:: bash

    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

Het subcommand ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` geeft de huidige URL van de lokale webserver terug.

.. note::

    De controller heeft geen naam voor de route, omdat deze nooit in de code wordt vermeld.

Groeperen op soortgelijke routes met een voorvoegsel
----------------------------------------------------

.. index::
    single: Annotations;@Route

De twee routes in de admincontroller hebben dezelfde ``/admin``-prefix. Schoon de routes op en configureer de prefix op de class zelf, in plaats van het op alle routes te herhalen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,9 @@ use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    +/**
    + * @Route("/admin")
    + */
     class AdminController extends AbstractController
     {
         private $twig;
    @@ -29,7 +32,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/comment/review/{id}", name="review_comment")
    +     * @Route("/comment/review/{id}", name="review_comment")
          */
         public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
         {
    @@ -58,7 +61,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     * @Route("/http-cache/{uri<.*>}", methods={"PURGE"})
          */
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
         {

CPU/geheugenintensieve handelingen cachen
-----------------------------------------

.. index::
    single: Process
    single: Components;Process

We hebben geen CPU of geheugenintensieve algoritmes op de website.  We gaan een commando maken die de huidige stap weergeeft waar we aan werken (de naam van de Git-tag van de huidige commit, om precies te zijn), zodat we kunnen gaan leren over *lokale caches*.

De Symfony Process-component stelt je in staat om een commando uit te voeren en het resultaat terug te krijgen (standaard- en foutmeldingen); installeer deze:

.. code-block:: bash

    $ symfony composer req process

Implementeer het commando:

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

    Je had ook ``make:command`` kunnen gebruiken om het commando te genereren:

    .. code-block:: bash
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Wat als we de output een paar minuten willen cachen? Gebruik de Symfony Cache:

.. code-block:: bash

    $ symfony composer req cache

En verpak de code in de cache logica:

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

Het proces wordt nu alleen aangeroepen als het ``app.current_step``-item niet in de cache zit.

Profileren en vergelijken van prestaties
----------------------------------------

Voeg nooit blindelings cache toe. Houd er rekening mee dat het toevoegen van caching een laag complexiteit toevoegt. En omdat we allemaal erg slecht zijn in het raden wat snel en wat traag zal zijn, kan het zijn dat je in een situatie terechtkomt waarin de cache jouw applicatie langzamer maakt.

Meet altijd de impact van het toevoegen van een cache met een profilertool zoals `Blackfire <https://blackfire.io/>`_.

Raadpleeg de stap over "Performance" om meer te weten te komen over hoe je Blackfire kan gebruiken om jouw code te testen voordat je deze deployed.

Een reverse proxy-cache configureren in productie
-------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: SymfonyCloud;Varnish
    single: Varnish

Gebruik Symfony's reverse proxy niet in productie. Geef altijd de voorkeur aan een reverse proxy zoals Varnish of een commerciële CDN op jouw infrastructuur.

Varnish toevoegen aan de SymfonyCloud-services:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -10,3 +10,12 @@ queue:
         type: rabbitmq:3.7
         disk: 1024
         size: S
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
    single: SymfonyCloud;Routes

Gebruik Varnish als eerste toegangspunt in de routes:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Maak ten slotte een ``config.vcl`` bestand aan om Varnish te configureren:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

ESI-ondersteuning inschakelen in Varnish
----------------------------------------

ESI-ondersteuning in Varnish moet voor elke request expliciet worden ingeschakeld. Om dit universeel te maken, gebruikt Symfony de standaard ``Surrogate-Capability`` en ``Surrogate-Control`` headers om over de ESI-ondersteuning te onderhandelen:

.. code-block:: vcl
    :caption: .symfony/config.vcl

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

De Varnish-cache legen
----------------------

Het invalideren van de cache zou op productie nooit noodzakelijk moeten zijn, behalve voor noodgevallen en misschien op branches buiten ``master``. Als je de cache vaak moet legen, betekent dit waarschijnlijk dat de cachingstrategie moet worden aangepast (door de TTL te verlagen of door een validatiestrategie te gebruiken in plaats van een vervalstrategie).

Hoe dan ook, laten we eens kijken hoe we Varnish moeten configureren voor cache-invalidatie:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
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

In het echte leven zou je in plaats daarvan waarschijnlijk IP's beperken, zoals beschreven in de `Varnish docs <https://varnish-cache.org/docs/trunk/users-guide/purging.html>`_.

Laten we nu een paar URL's legen:

.. code-block:: bash

    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`
    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`conference_header

De URL's zien er een beetje vreemd uit omdat de URL's die worden teruggestuurd door ``env:urls`` al eindigen met ``/``.

.. sidebar:: Verder gaan

    * `Cloudflare <https://www.cloudflare.com>`_, het wereldwijde cloudplatform;

    * `Varnish HTTP-Cachedocs <https://varnish-cache.org/docs/index.html>`_;

    * `ESI-specificatie <https://www.w3.org/TR/esi-lang>`_ en `ESI-ontwikkelaarsmiddelen <https://www.akamai.com/us/en/support/esi.jsp>`_;

    * `HTTP-cache validatiemodel <https://symfony.com/doc/current/http_cache/validation.html>`_;

    * `HTTP Cache in SymfonyCloud <https://symfony.com/doc/current/cloud/cookbooks/cache.html>`_.

.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
