Cache pentru performanță
==========================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Problemele de performanță ar putea veni odată cu creșterea popularității. Câteva exemple tipice: indici de date lipsă sau solicitări SQL sporite per pagină. Nu vei avea probleme cu o bază de date goală, dar cu mai mult trafic și volum de date în creștere, ar putea apărea la un moment dat.

Adăugarea anteturilor de cache HTTP
------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Utilizarea strategiilor de cache HTTP este o modalitate excelentă de a maximiza performanțele pentru utilizatorii finali cu puțin efort. Adaugă un proxy cache în producție pentru a activa memorarea cache-ului și folosește un `CDN`_ pentru a utiliza un cache periferic pentru o performanță și mai bună.

Hai să memorăm în cache pagina principală pentru o oră:

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

Metoda ``setSharedMaxAge()`` configurează expirarea cache-ului pentru proxy-ul revers. Utilizează ``setMaxAge()`` pentru a controla memoria cache a browserului. Timpul este exprimat în secunde (1 oră = 60 minute = 3600 secunde).

Memorarea în cache a paginii conferinței este mai dificilă, dar și mai dinamică. Oricine poate adăuga un comentariu oricând și nimeni nu vrea să aștepte o oră pentru a-l vedea online. În astfel de cazuri, utilizează strategia de *validare HTTP*.

Activarea nucleului cache HTTP Symfony
--------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Pentru a testa strategia de cache HTTP, utilizează reverse proxy-ul pus la dispoziție de componenta Symfony HTTP:

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

Pe lângă faptul că este un proxy revers HTTP cu funcții complete, proxy-ul revers Symfony HTTP (prin clasa ``HttpCache``) adaugă câteva informații de depanare utile, de exemplu anteturi HTTP. Acest lucru ajută foarte mult la validarea anteturilor cache-ului pe care le-am setat.

Verifică-l pe pagina principală:

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

Pentru prima solicitare, serverul de memorie cache îți spune că a eșuat (``miss``) și că a efectuat o stocare (``store``) pentru a memora în cache răspunsul. Verifică antetul ``cache-control`` pentru a vedea strategia de memorie cache configurată.

Pentru solicitări ulterioare, răspunsul se află în cache (``age`` - timpul de expirare - fiind de asemenea actualizat):

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

Evitarea cererilor SQL cu ESI
-----------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Ascultătorul ``TwigEventSubscriber`` injectează o variabilă globală în Twig cu toate obiectele conferinței. Face acest lucru pentru fiecare pagină a site-ului. Este probabil o țintă excelentă pentru optimizare.

Nu vei adăuga noi conferințe în fiecare zi, astfel încât codul interogează mereu exact aceleași date din baza de date.

S-ar putea să dorim să memorăm în cache numele și identificatorilor de cale a conferinței cu Symfony Cache, dar de câte ori este posibil, îmi place să mă bazez pe infrastructura de memorie cache HTTP.

Când dorești să memorezi în cache un fragment dintr-o pagină, mută-l în afara cererii HTTP curente, creând un *sub-request*. *ESI* (Edge Side includes) este perfect în acest caz. Un ESI este o modalitate de a încorpora rezultatul unei solicitări HTTP în cadrul altei solicitări.

Creează un controler care returnează doar fragmentul HTML care afișează conferințele:

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

Creează șablonul corespunzător:

.. code-block:: twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Execută ``/conference_header`` pentru a te asigura că totul funcționează corect.

.. index::
    single: Twig;render
    single: Twig;path

E timpul să dezvălui trucul! Actualizează aspectul Twig pentru a apela controlerul pe care tocmai l-am creat:

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

Și *voilà*. Actualizează pagina și site-ul afișează în continuare aceeași lucru.

.. tip::

    Folosește panoul de profilare Symfony „Request / Response” pentru a afla mai multe despre cererea principală și sub-cererile sale.

Acum, de fiecare dată când accesezi o pagină din browser, sunt executate două solicitări HTTP, una pentru antet și una pentru pagina principală. Ai înrăutățit performanța. Felicitări!

Apelul HTTP pentru conferință este realizat în prezent de Symfony, astfel încât nu este implicată niciun transfer HTTP. Acest lucru înseamnă, de asemenea, că nu există nicio modalitate de a beneficia de anteturile de cache HTTP.

Convertește apelul într-unul „real” HTTP folosind un ESI.

În primul rând, activează suportul ESI:

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

Apoi, utilizează ``render_esi`` în loc de ``render``:

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

Dacă Symfony detectează un proxy revers care știe cum să facă față ESI-urilor, acesta permite asistența în mod automat (dacă nu, revine la redarea cererii în mod sincron).

Deoarece reverse proxy-ul Symfony acceptă ESI-urile, să verificăm jurnalele sale (înlătură mai întâi memoria cache - consultă secțiunea „Purging” mai jos):

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

Actualizează de câteva ori: răspunsul ``/`` este memorat în cache iar ``/conference_header`` nu este. Am realizat ceva grozav: să avem întreaga pagină în cache, dar să avem în continuare o parte dinamică.

Totuși, nu asta ne dorim. Salvăm pagina antet pentru o oră în cache, independent de orice altceva:

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

Cache este acum activat pentru ambele solicitări:

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

Antetul ``x-symfony-cache`` conține două elemente: cererea principală ``/`` și o sub-cerere (ESI ``conference_header``). Ambele sunt în cache (``fresh``).

Strategia cache poate fi diferită de pagina principală și de ESI-urile sale. Dacă avem o pagină „about”, s-ar putea să vrem să o depozităm timp de o săptămână în memoria cache și să avem totuși antetul actualizat la fiecare oră.

Înlătură ascultătorul deoarece nu mai avem nevoie de el:

.. code-block:: bash

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Curățirea cache-ului HTTP pentru testare
------------------------------------------

Testarea site-ului web într-un browser sau prin teste automate devine ceva mai dificilă cu un strat de memorie în cache.

You can manually remove all the HTTP cache by removing the
``var/cache/dev/http_cache/`` directory:

.. code-block:: bash

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Annotations;@Route

Această strategie nu funcționează bine dacă dorești să invalidezi anumite adrese URL sau dacă dorești să integrezi invalidarea cache-ului în testele funcționale. Să adăugăm un punct de referință HTTP mic, admin pentru a invalida unele adrese URL:

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

Noul controler a fost restricționat la metoda HTTP ``PURGE``. Această metodă nu este în standardul HTTP, dar este utilizată pe scară largă pentru a invalida cache-urile.

În mod implicit, parametrii rutelor nu pot conține ``/``, deoarece separă segmentele URL. Poți înlocui această restricție pentru ultimul parametru de rută, precum ``uri``, prin setarea propriului model de cerință (``.*``).

Modul în care obținem instanța ``HttpCache`` poate arăta și un pic ciudat; folosim o clasă anonimă, deoarece accesarea celei „reale” nu este posibilă. Instanța ``HttpCache`` înfășoară nucleul real, care nu cunoaște stratul cache așa cum ar trebui să fie.

Invalidează pagina principală și antetul conferinței prin următoarele apeluri cURL:

.. code-block:: bash

    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

Subcomanda ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` returnează adresa URL curentă a serverului web local.

.. note::

    Controlerul nu are un nume de rută, întrucât nu va fi niciodată menționat în cod.

Gruparea rutelor similare cu un prefix
--------------------------------------

.. index::
    single: Annotations;@Route

Cele două rute din controlerul admin au același prefix ``/admin``. În loc să-l repeți pe toate rutele, refactorizează rutele pentru a configura prefixul pe clasa în sine:

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

Salvarea în cache a operațiunilor intensive de CPU/Memorie
------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

Nu avem algoritmi cu consum intensiv a resurselor CPU sau memorie pe site. Pentru a vorbi despre *cache-urile locale*, să creăm o comandă care să afișeze pasul curent la care lucrăm (pentru a fi mai exact, numele tag-ului Git atașat la versiunea locală Git).

Componenta Symfony Process îți permite să rulezi o comandă și să obții rezultatul înapoi (ieșire standard și eroare); instalează-l:

.. code-block:: bash

    $ symfony composer req process

Implementează comanda:

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

    Ai fi putut folosi ``make: command`` pentru a crea comanda:

    .. code-block:: bash
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Ce se întâmplă dacă vrem să memorăm în cache ieșirea timp de câteva minute? Folosește memoria Symfony Cache:

.. code-block:: bash

    $ symfony composer req cache

Și înfășoară codul cu logica cache:

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

Procesul este acum apelat doar dacă elementul ``app.current_step`` nu se află în memoria cache.

Profilarea și compararea performanței
---------------------------------------

Nu adăuga niciodată cache-ul orbește. Reține că adăugarea unor cache adaugă un strat de complexitate. Și cum toți suntem foarte răi în ghicirea ce va fi rapid și ce este lent, s-ar putea să ajungi într-o situație în care cache-ul face ca aplicația să fie mai lentă.

Măsoară întotdeauna impactul adăugării unei memorii cache cu un instrument de profilare precum `Blackfire <https://blackfire.io/>`_.

Consultă pasul despre „Performanță” pentru a afla mai multe despre modul în care poți utiliza Blackfire pentru a testa codul tău înainte de implementare.

Configurarea unui cache proxy revers pentru producție
------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: SymfonyCloud;Varnish
    single: Varnish

Nu folosi proxy-ul Symfony revers în producție. Preferă întotdeauna un proxy revers ca Varnish pe infrastructura ta sau pe un CDN comercial.

Adaugă Varnish pentru serviciile SymfonyCloud:

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

Utilizează Varnish ca principal punct de intrare în rute:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

În final, creează un fișier ``config.vcl`` pentru a configura Varnish:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Activarea suportului ESI pe Varnish
-----------------------------------

Asistența ESI pe Varnish ar trebui să fie activată explicit pentru fiecare solicitare. Pentru a-l face universal, Symfony folosește anteturile standard ``Surrogate-Capability`` și ``Surrogate-Control`` pentru a negocia suportul ESI:

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

Curățirea cache-ului Varnish
------------------------------

Invalidarea cache-ului în producție ar trebui probabil să nu fie necesară niciodată, cu excepția scopurilor de urgență și poate pe ramurile care nu sunt ``master``. Dacă e nevoie să elimini deseori memoria cache, înseamnă probabil că trebuie modificată strategia de cache (scăzând TTL sau folosind o strategie de validare în loc de una de expirare).

Oricum, hai să vedem cum poți configura Varnish pentru invalidarea cache-ului:

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

În viața reală, probabil că restricționezi prin IP-uri, în locul metodelor descrise în secțiunea „documentației Varnish <https://varnish-cache.org/docs/trunk/users-guide/purging.html>`_.

Curăță câteva adrese URL acum:

.. code-block:: bash

    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`
    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`conference_header

Adresele URL arată un pic ciudat, deoarece adresele URL returnate de ``env:urls`` se termină deja cu ``/``.

.. sidebar:: Mergând mai departe

    * `Cloudflare <https://www.cloudflare.com>`_, platforma globală de cloud;

    * `Documentația de cache HTTP Varnish <https://varnish-cache.org/docs/index.html>`_;

    * `Specificația ESI <https://www.w3.org/TR/esi-lang>`_ și `resurse dezvoltator ESI <https://www.akamai.com/us/en/support/esi.jsp>`_ ;

    * `Model de validare cache HTTP <https://symfony.com/doc/current/http_cache/validation.html>`_;

    * `HTTP Cache in SymfonyCloud <https://symfony.com/doc/current/cloud/cookbooks/cache.html>`_.

.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
