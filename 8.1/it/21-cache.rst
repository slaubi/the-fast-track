Caching per le prestazioni
==========================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

I problemi di prestazioni sono una conseguenza della popolarità. Alcuni esempi tipici: indici mancanti nelle tabelle del database o troppe query SQL per pagina. Non avrete problemi di prestazioni con un database vuoto, ma potreste averne all'aumentare del traffico e dei dati.

Header HTTP per la cache
------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

L'utilizzo di strategie di cache HTTP è un ottimo modo per migliorare le prestazioni con poco sforzo. Potete aggiungete un reverse proxy in produzione per abilitare la cache e usare un `CDN`_ per sfruttare una rete distribuita di cache, per prestazioni ancora migliori.

Mettiamo in cache l'homepage per un'ora:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -14,6 +14,7 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\Cache;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    @@ -27,6 +28,7 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {

L'attributo ``#[Cache]`` configura la scadenza della cache per i reverse proxy tramite il suo argomento ``smaxage``; utilizzare ``maxage`` per impostare il tempo di cache per i browser. Il tempo è espresso in secondi (1 ora = 60 minuti = 3600 secondi). E come per il routing o il rate limiting, la politica di cache viene dichiarata proprio dove si applica: sul controller.

Gestire la cache della pagina della conferenza è più impegnativo, perché è una pagina più dinamica. Chiunque può aggiungere un commento in qualsiasi momento e nessuno vuole aspettare un'ora per vederlo online. In questi casi, utilizzare la strategia di *validazione HTTP* .

Attivare la cache HTTP nel kernel di Symfony
--------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Per testare la strategia di cache HTTP, occorre abilitare il reverse proxy HTTP di Symfony, ma solo in ambiente di "sviluppo" (per l'ambiente di "produzione" utilizzeremo un soluzione "più robusta"):

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Oltre a essere un vero e proprio reverse proxy HTTP, il reverse proxy HTTP di Symfony (tramite la libreria ``HttpCache``) aggiunge alcune informazioni di debug negli header HTTP. Questo aiuta durante la validazione degli header per la cache che abbiamo impostato.

Controlliamo l'homepage:

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

Durante la prima richiesta, il server di cache ci informa che non ha trovato la risposta nella cache (``miss``) e ha memorizzato (``store``) la risposta nella cache. Controllare l'intestazione HTTP chiamata ``cache-control`` per vedere la strategia di cache configurata.

Nelle successive richieste, le risposte vengono prese dalla cache (anche l'header HTTP ``age`` è stato aggiornato):

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

Evitare richieste SQL con ESI
-----------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Il listener ``TwigEventListener`` aggiunge una variabile globale in Twig con tutti gli oggetti conferenza. Questo viene fatto per ogni singola pagina del sito. Probabilmente dovrebbe essere ottimizzato.

Non si aggiungono nuove conferenze ogni giorno, quindi il codice interroga gli stessi identici dati dal database più e più volte.

Potremmo voler mettere in cache i nomi e gli slug delle conferenze con la cache di Symfony, ma quando possibile mi piace fare affidamento sull'infrastruttura di cache HTTP.

Quando vogliamo mettere in cache un frammento di una pagina, lo spostiamo al di fuori della richiesta HTTP corrente creando una *sotto-richiesta* . Per questo caso d'uso *ESI* è la soluzione perfetta. ESI è un modo per incorporare il risultato di una richiesta HTTP in un'altra.

Creiamo un controller che restituisce solo il frammento HTML che visualizza le conferenze:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,14 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(

Creiamo il template corrispondente:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Apriamo ``/conference_header`` per controllare che tutto funzioni correttamente.

.. index::
    single: Twig;render
    single: Twig;path

È ora di svelare il trucco! Aggiorniamo il layout Twig per chiamare il controller che abbiamo appena creato:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
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

E *voilà*. Aggiorniamo la pagina e il sito continuerà a mostrare le stesse informazioni.

.. tip::

    Usiamo il pannello "Request / Response" del Profiler di Symfony per saperne di più sulla richiesta principale e sulle sue sotto-richieste.

Ora, ogni volta che si raggiunge una pagina nel browser vengono eseguite due richieste HTTP: una per l'intestazione e una per la pagina principale. Abbiamo peggiorato le prestazioni. Congratulazioni!

La chiamata HTTP all'intestazione della conferenza è attualmente effettuata internamente da Symfony, quindi non è previsto alcun round-trip HTTP. Questo significa anche che non c'è modo di beneficiare degli header della cache HTTP.

Convertire la chiamata in una chiamata HTTP "reale" utilizzando ESI.

In primo luogo, attiviamo il supporto ESI:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
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

Quindi, utilizziamo ``render_esi`` al posto di ``render``:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Se Symfony rileva un reverse proxy che sa come trattare gli ESI, abilita automaticamente il supporto (in caso contrario, ritorna al render sincrono della richiesta secondaria).

Poiché il reverse proxy di Symfony supporta ESI, controlliamo i suoi log (rimuoviamo prima la cache, si veda la sezione "Pulizia" più avanti):

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

Aggiorniamo qualche volta la pagina: la risposta relativa al percorso ``/`` è salvata in cache, mentre quella relativa al percorso ``/conference_header`` non lo è. Abbiamo ottenuto un grande risultato: l'intera pagina è in cache, ma una sua parte è ancora dinamica.

Ma non è quello che vogliamo. Manteniamo in cache la pagina di intestazione per un'ora, indipendentemente da tutto il resto:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {

La cache è ora abilitata per entrambe le richieste:

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

L'intestazione ``x-symfony-cache`` contiene due elementi: la richiesta principale ``/`` e una richiesta secondaria (l'ESI ``conference_header``). Entrambi sono in cache (``fresh``).

La strategia di cache può essere diversa tra la pagina principale e i suoi ESI. Se abbiamo una pagina "about", potremmo volerla conservare per una settimana in cache ma avere comunque l'intestazione aggiornata ogni ora.

Rimuoviamo il listener, non ne abbiamo più bisogno:

.. code-block:: terminal

    $ rm src/EventListener/TwigEventListener.php

Pulire la cache HTTP per i test
-------------------------------

Testare il sito in un browser o tramite test automatici diventa un po' più difficile quando c'è un livello di cache.

È possibile rimuovere manualmente tutta la cache HTTP, svuotando la cartella ``var/cache/dev/http_cache/``:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

Questa strategia non funziona bene se si vogliono invalidare solo alcuni URL o se si vuole integrare l'invalidazione della cache nei test funzionali. Aggiungiamo un piccolo endpoint HTTP, solo per l'amministrazione, per invalidare alcuni URL:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
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

Il nuovo controller è stato limitato al metodo HTTP ``PURGE``. Questo metodo non fa parte dello standard HTTP, ma è ampiamente usato per invalidare le cache.

Per impostazione predefinita, i parametri della rotta non possono contenere ``/``, visto che funge da separatore negli URL. È possibile sovrascrivere questa restrizione per l'ultimo parametro della rotta, come ``uri``, impostando lo schema dei requisiti (``.*``).

Il modo in cui otteniamo l'istanza di ``HttpCache`` può anche sembrare un po' strano; stiamo usando una classe anonima perché l'accesso a quella "reale" non è possibile. L'istanza di ``HttpCache`` avvolge il kernel reale che, come è giusto che sia, non è a conoscenza del livello di cache.

Invalidiamo la homepage e l'intestazione della conferenza tramite le seguenti chiamate cURL:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

Il comando ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` restituisce l'URL corrente del server web locale.

.. note::

    Il controller non ha un nome di rotta, in quanto non sarà mai menzionato nel codice.

Disabilitare la cache HTTP in sviluppo
--------------------------------------

La cache HTTP è stata preziosa per convalidare i nostri header di cache e per imparare a eliminare le voci obsolete. Ma tenere un reverse proxy attivo nell'ambiente di sviluppo è insolito, e diventa presto d'intralcio: le risposte vengono servite dalla cache mentre si itera sul codice, e alcuni asset dei vendor vengono perfino serviti con un corpo vuoto a causa di una vecchia limitazione di HttpCache con le risposte di tipo file.

Ora che tutto è convalidato, disabilitarla; Varnish prenderà il suo posto in produzione:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -14,7 +14,3 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    -
    -when@dev:
    -    framework:
    -        http_cache: true

Raggruppamento di percorsi simili con un prefisso
-------------------------------------------------

.. index::
    single: Attributes;Route

Le due rotte del controller di amministrazione hanno lo stesso prefisso ``/admin``. Invece di ripeterlo su tutte le rotte, rifattorizziamole in modo da configurare il prefisso sulla classe stessa:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

Cache di operazioni che richiedono molta CPU o memoria
------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

Non abbiamo algoritmi che impegnino molta CPU o memoria in questo sito. Per parlare di *cache locali* , creeremo un comando che mostra il passo su cui stiamo lavorando (per essere più precisi, il nome del tag di git associato al commit corrente).

Il componente Process di Symfony permette di eseguire un comando e ottenerne il risultato (standard output ed error output).

Implementiamo il comando:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand
    {
        public function __invoke(OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Cache
    single: Components;Cache

E se volessimo mettere in cache l'output per qualche minuto? Usiamo il componente Cache di Symfony.

Symfony inietta i servizi tipizzati nel metodo ``__invoke()`` di un comando, allo stesso modo degli argomenti dei controller. Aggiungiamo la logica della cache al codice:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -6,15 +6,21 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand
     {
    -    public function __invoke(OutputInterface $output): int
    +    public function __invoke(OutputInterface $output, CacheInterface $cache): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

Il processo è ora richiamato solo se l'elemento ``app.current_step`` non è in cache.

Profilazione e confronto delle prestazioni
------------------------------------------

Evitiamo di aggiungere cache alla cieca. Teniamo presente che l'aggiunta di cache aggiunge uno strato di complessità e, dato che siamo tutti pessimi nell'indovinare cosa sarà veloce e cosa sarà lento, potremmo ritrovarci in una situazione in cui la cache rallenti la nostra applicazione.

Misuriamo sempre l'impatto dell'aggiunta di una cache con uno strumento di profilazione come `Blackfire`_.

Fare riferimento al passo "Prestazioni" per saperne di più su come utilizzare Blackfire per testare il codice prima del deploy.

Configurazione di una cache di reverse proxy in produzione
----------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

Invece che utilizzare il reverse proxy di Symfony in produzione, utilizzero Varnish che è un reverse proxy "più robusto".

Aggiungere Varnish ai servizi di Upsun:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -6,6 +6,15 @@ services:
         database:
             type: postgresql:16

    +    varnish:
    +        type: varnish:9.0
    +        relationships:
    +            application: 'app:http'
    +        configuration:
    +            vcl: !include
    +                type: string
    +                path: config.vcl
    +
     applications:

.. index::
    single: Upsun;Routes

Utilizzare Varnish come punto di ingresso principale nelle rotte:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

Infine, creiamo un file di configurazione ``config.vcl`` per Varnish:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Abilitare il supporto a ESI su Varnish
--------------------------------------

Il supporto ESI su Varnish dovrebbe essere abilitato esplicitamente per ogni richiesta. Per renderlo universale, Symfony usa lo standard ``Surrogate-Capability`` e gli header ``Surrogate-Control`` per negoziare il supporto ESI:

.. code-block:: vcl
    :caption: .upsun/config.vcl

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

Pulire la cache di Varnish
--------------------------

L'invalidazione della cache in produzione probabilmente non dovrebbe mai essere necessaria, tranne che per scopi di emergenza e forse su branch diversi da ``master``. Se è necessario pulire spesso la cache, probabilmente significa che la strategia di cache dovrebbe essere ottimizzata (abbassando il TTL o usando una strategia di validazione invece di una strategia di scadenza).

In ogni caso, vediamo come configurare Varnish per invalidare la cache:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
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

Nella vita reale, probabilmente utilizzeremo limitazioni sulla base dell'indirizzo IP, come descritto nella `documentazione di Varnish`_.

Ora invalidiamo alcuni URL:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

Gli URL sembrano un po' strani, poiché quelli restituiti da ``env:url`` finiscono già con ``/``.

.. sidebar:: Andare oltre

    * `Cloudflare`_, la piattaforma cloud globale;

    * `Documentazione sulla cache HTTP di Varnish`_;

    * `Specifiche ESI`_ e `risorse per gli sviluppatori ESI`_;

    * `Modello di validazione della cache HTTP`_;

    * `Cache HTTP in Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`documentazione di Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Documentazione sulla cache HTTP di Varnish`: https://varnish-cache.org/docs/index.html
.. _`Specifiche ESI`: https://www.w3.org/TR/esi-lang
.. _`risorse per gli sviluppatori ESI`: https://www.akamai.com/us/en/support/esi.jsp
.. _`Modello di validazione della cache HTTP`: https://symfony.com/doc/current/http_cache/validation.html
.. _`Cache HTTP in Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
