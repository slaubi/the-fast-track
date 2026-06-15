Użycie pamięci podręcznej w celu zwiększenia wydajności
============================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Problemy z wydajnością mogą pojawić się wraz ze wzrostem popularności aplikacji. Typowe problemy: nie zrobiłeś indeksów baz danych, a ze strony idzie grad zapytań SQL. Z pustą bazą danych nie będziesz miał żadnych problemów, ale mogą się one pojawić wraz z większym natężeniem ruchu i wzrostem ilości danych.

Dodawanie nagłówków pamięci podręcznej HTTP (ang. HTTP cache headers)
--------------------------------------------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Wykorzystanie strategii użycia pamięci podręcznej HTTP to świetny sposób na maksymalizację wydajności dla użytkowników końcowych przy niewielkim nakładzie pracy. Dodaj zwrotny serwer pośredniczący pamięci podręcznej (ang. reverse proxy cache) w środowisku produkcyjnym oraz użyj `CDN`_ do buforowania na węźle krańcowym dla jeszcze lepszej wydajności.

Zapiszmy stronę domową w pamięci podręcznej na godzinę:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -30,7 +30,7 @@ class ConferenceController extends AbstractController
         {
             return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Metoda ``setSharedMaxAge()`` konfiguruje wygaśnięcie pamięci podręcznej dla zwrotnego serwera pośredniczącego (ang. reverse proxy). Użyj ``setMaxAge()`` do kontrolowania pamięci podręcznej przeglądarki. Czas wyrażany jest w sekundach (1 godzina = 60 minut = 3600 sekund).

Buforowanie strony konferencji jest wyzwaniem, ponieważ jest ona bardzo dynamiczna. Każdy może dodać komentarz w dowolnej chwili i nikt nie chce czekać godzinę, aby zobaczyć go online. W takich przypadkach należy stosować strategię *walidacji HTTP*.

Aktywacja Symfony HTTP Cache Kernel
-----------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Aby przetestować strategię pamięci podręcznej HTTP, włącz zwrotny serwer pośredniczący (ang. reverse proxy) Symfony HTTP, ale tylko dla środowiska "development" (dla środowiska "production" użyjemy bardziej "wydajnego" rozwiązania):

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -23,3 +23,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Oprócz tego, że jest to pełnoprawny zwrotny serwer pośredniczący (ang. reverse proxy)  HTTP, dodaje on (poprzez klasę ``HttpCache``) kilka przydatnych informacji o debugowaniu jako nagłówki HTTP. Bardzo pomaga to w sprawdzaniu poprawności nagłówków pamięci podręcznej, które ustawiliśmy.

Możesz sprawdzić jego działanie na stronie głównej:

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

W przypadku pierwszego żądania, serwer pamięci podręcznej mówi, że był to ``miss`` (brak wpisu w pamięci podręcznej) i że wykonał akcję ``store`` (buforowania odpowiedzi). Sprawdź nagłówek ``cache-control`` (kontrola pamięci podręcznej), aby zobaczyć konfigurację odpowiedzialną za strategię pamięci podręcznej.

W przypadku kolejnych żądań odpowiedź jest przechowywana w pamięci podręcznej. Również ``age`` (czas, który upłynął od ostatniego zapisu) został zaktualizowany:

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

Unikanie zapytań SQL za pomocą ESI
------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Nasłuchiwacz (ang. listener) ``TwigEventSubscriber`` wstrzykuje globalną zmienną do Twiga ze wszystkimi obiektami konferencji. Czyni to dla każdej strony witryny. Jest to prawdopodobnie świetne miejsce do optymalizacji.

Nie będziesz dodawał nowych konferencji codziennie, więc kod odpytuje o dokładnie te same dane z bazy danych w kółko.

Możemy chcieć buforować nazwy konferencji i slugi użwajac Symfony Cache, ale, kiedy tylko jest to możliwe, lubię polegać na infrastrukturze buforowania HTTP.

Jeśli chcesz zapisać w pamięci podręcznej fragment strony, przenieś go poza bieżące żądanie HTTP, tworząc *żądanie cząstkowe*. *ESI* jest do tego idealnym rozwiązaniem. ESI jest sposobem osadzenia wyniku żądania HTTP w innym żądaniu.

Utwórz kontroler zwracający tylko fragment kodu HTML, który wyświetla konferencje:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -33,6 +33,14 @@ class ConferenceController extends AbstractController
             ])->setSharedMaxAge(3600);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,

Utwórz odpowiedni szablon:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Otwórz ``/conference/header`` aby sprawdzić, czy wszystko działa poprawnie.

.. index::
    single: Twig;render
    single: Twig;path

Czas na magiczną sztuczkę! Zaktualizuj szablon Twig, aby wywołać kontroler, który właśnie stworzyliśmy:

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

*Et voilà*. Odśwież stronę, a witryna nadal wyświetla się tak samo.

.. tip::

    Użyj panelu "Request / Response" profilera Symfony, aby dowiedzieć się więcej o głównym żądaniu i żądaniach cząstkowych.

Teraz za każdym razem, gdy wejdziesz na stronę w przeglądarce, wykonywane są dwa żądania HTTP, jedno dla nagłówka i jedno dla strony głównej. Pogorszyliśmy wydajność. Gratulacje!

Wywołanie HTTP nagłówka konferencji jest obecnie wykonywane wewnętrznie przez Symfony, dzięki czemu unikamy zewnętrznego połączenia HTTP. Oznacza to również, że nie ma możliwości skorzystania z nagłówków pamięci podręcznej HTTP.

Skonwertuj połączenie na "prawdziwe" połączenie HTTP za pomocą ESI.

Po pierwsze, włącz obsługę ESI:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -13,7 +13,7 @@ framework:
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

Następnie użyj ``render_esi`` zamiast ``render``:

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

Jeśli Symfony wykryje zwrotny serwer pośredniczący (ang. reverse proxy), który wie, jak radzić sobie z ESI – włącza obsługę automatycznie. Jeśli nie, to wraca do renderowania żądania cząstkowego synchronicznie.

Ponieważ zwrotny serwer pośredniczący (ang. reverse proxy)  Symfony obsługuje ESI, sprawdźmy jego logi (najpierw wyczyść pamięć podręczną – zobacz "Usuwanie" poniżej):

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

Odśwież kilka razy: odpowiedź ``/`` jest buforowana, a ``/conference_header`` już nie. Osiągnęliśmy coś wspaniałego: mamy całą stronę w pamięci podręcznej, ale wciąż jedna z jej części jest dynamiczna.

Nie tego jednak chcemy. Zapisz stronę z nagłówkiem w pamięci podręcznej na godzinę, niezależnie od wszystkiego innego:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -38,7 +38,7 @@ class ConferenceController extends AbstractController
         {
             return $this->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Pamięć podręczna jest teraz włączona dla obu żądań:

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

Nagłówek ``x-symfony-cache`` zawiera dwa elementy: żądanie główne ``/`` oraz żądanie cząstkowe (``conference_header`` ESI). Oba znajdują się w pamięci podręcznej (``fresh``).

Strategia pamięci podręcznej może być odmienna dla strony głównej i jej ESI. Jeśli mamy stronę "o nas", możemy chcieć ją przechować przez tydzień w pamięci podręcznej i nadal mieć nagłówek aktualizowany co godzinę.

Usuń nasłuchiwacz (ang. listener), bo już go nie potrzebujemy:

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Oczyszczanie pamięci podręcznej HTTP na potrzeby testów
----------------------------------------------------------

Testowanie strony internetowej w przeglądarce lub za pomocą testów automatycznych staje się nieco trudniejsze w przypadku warstwy pamięci podręcznej.

Możesz ręcznie wyczyścić całą pamięć podręczną HTTP usuwając katalog ``var/cache/dev/http_cache/``:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

To podejście nie działa za dobrze, jeśli chcesz tylko unieważnić niektóre adresy URL lub jeśli chcesz włączyć unieważnienie pamięci podręcznej do testów funkcjonalnych. Dodajmy mały, dostępny tylko dla konta administracyjnego, punkt końcowy HTTP, aby unieważnić niektóre adresy URL:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -17,6 +17,8 @@ security:
                 lazy: true
                 provider: app_user_provider
                 custom_authenticator: App\Security\AppAuthenticator
    +            http_basic: { realm: Admin Area }
    +            entry_point: App\Security\AppAuthenticator
                 logout:
                     path: app_logout
                     # where to redirect after logout
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

Nowy kontroler został ograniczony do metody ``PURGE`` HTTP. Metoda ta nie znajduje się w standardzie HTTP, ale jest powszechnie stosowana do unieważniania pamięci podręcznej.

Domyślnie parametry trasy nie mogą zawierać ``/``, ponieważ rozdzielają segmenty URL. Ograniczenie to można pominąć w odniesieniu do ostatniego parametru trasy, na przykład ``uri``. ustawiając własny wzór wymagań (``.*``).

Sposób w jaki otrzymujemy instancję ``HttpCache`` może wyglądać nieco dziwnie; używamy klasy anonimowej, ponieważ dostęp do klasy "rzeczywistej" nie jest możliwy. Instancja ``HttpCache`` owija prawdziwe jądro, które nie jest świadome warstwy pamięci podręcznej, tak jak powinno być.

Unieważnienie pamięci podręcznej strony głównej i nagłówka konferencji za pośrednictwem następujących połączeń cURL:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

Podpolecenie ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` zwraca bieżący adres URL lokalnego serwera WWW.

.. note::

    Kontroler nie posiada nazwy trasy, ponieważ nigdy nie będziemy się do niego odwoływać w kodzie.

Grupowanie podobnych tras z użyciem prefiksu
---------------------------------------------

.. index::
    single: Attributes;Route

Dwie trasy w kontrolerze panelu administracyjnego mają ten sam prefiks ``/admin``. Zamiast powtarzać go na wszystkich trasach, należy zrefaktorować trasy w celu skonfigurowania prefiksu na samej klasie:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Annotation\Route;
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

Zapisywanie do pamięci podręcznej operacji obciążających procesor/pamięć
-------------------------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

Nie mamy na stronie internetowej algorytmów intensywnie wykorzystujących procesor czy pamięć. Aby porozmawiać o *lokalnej pamięci podręcznej*, stwórzmy polecenie, które wyświetla informację o etapie, nad którym pracujemy, a dokładniej mówiąc, nazwę tagu Git dołączoną do aktualnego zatwierdzenia (ang. commit) Gita.

Komponent Symfony Process pozwala uruchomić polecenie i otrzymać wynik jego działania (dane ze standardowego strumienia wyjścia i standardowego strumienia błędów).

Zaimplementuj polecenie:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand extends Command
    {
        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Możesz użyć ``make:command`` do stworzenia polecenia:

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

A jeśli chcemy przechowywać dane wyjściowe przez kilka minut? Dodaj do projektu moduł Symfony Cache.

I użyj pamięci podręcznej:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
    @@ -7,15 +7,27 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand extends Command
     {
    +    public function __construct(
    +         private CacheInterface $cache,
    +    ) {
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

             return Command::SUCCESS;
         }

Proces jest teraz wywoływany tylko wtedy, gdy element ``app.current_step`` nie znajduje się w pamięci podręcznej.

Profilowanie i porównywanie wydajności
----------------------------------------

Nigdy nie dodawaj pamięci podręcznej bez zastanowienia. Należy pamiętać, że dodanie pamięci podręcznej zwiększa poziom złożoności. A ponieważ źle nam idzie zgadywanie, co będzie szybkie, a co powolne, możesz znaleźć się w sytuacji, w której pamięć podręczna spowolni Twoją aplikację.

Zawsze zmierz wpływ dodawania pamięci podręcznej za pomocą narzędzia do profilowania, takiego jak `Blackfire`_.

Zapoznaj się z etapem "Wydajność", aby dowiedzieć się więcej o tym, jak można użyć narzędzia Blackfire do przetestowania kodu przed wdrożeniem.

Konfiguracja  zwrotnego serwera pośredniczącego pamięci podręcznej (ang. reverse proxy cache) w środowisku produkcyjnym
----------------------------------------------------------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

Zamiast używać zwrotnego proxy Symfony na produkcji, użyjemy bardziej "wydajnego" zwrotnego proxy Varnish.

Dodaj Varnish do usług Upsun:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -2,3 +2,12 @@
     database:
         type: postgresql:14
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
    single: Upsun;Routes

Użyj serwera Varnish jako głównego punktu wejścia na trasach:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/routes.yaml
    +++ b/.platform/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Na koniec, utwórz plik ``config.vcl`` do konfiguracji Varnish:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Włączenie wsparcia ESI na serwerze Varnish
--------------------------------------------

Obsługa ESI na Varnish powinna być włączona bezpośrednio dla każdego żądania. W celu uniwersalizacji Symfony wykorzystuje standardowe nagłówki ``Surrogate-Capability`` i ``Surrogate-Control`` do negocjowania wsparcia ESI:

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

Oczyszczanie (ang. purging) pamięci podręcznej Varnish.
---------------------------------------------------------

Unieważnienie pamięci podręcznej w środowisku produkcyjnym prawdopodobnie nigdy nie będzie potrzebne, z wyjątkiem celów awaryjnych i raczej na gałęziach innych niż ``master``. Jeśli musisz często czyścić pamięć podręczną, prawdopodobnie oznacza to, że strategia użycia pamięci podręcznej powinna zostać poprawiona (poprzez obniżenie TTL lub poprzez zastosowanie strategii walidacji zamiast strategii wygaśnięcia).

W każdym razie, zobaczmy jak skonfigurować serwer Varnish do unieważnienia pamięci podręcznej:

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

W rzeczywistości, prawdopodobnie należałoby zastosować ograniczenie po IP,  jak to opisano w `dokumentacji Varnish`_.

Oczyść (ang. purge) teraz kilka adresów URL:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

Adresy URL wyglądają nieco dziwnie, ponieważ adresy zwracane przez ``env:url`` już kończą się na ``/``.

.. sidebar:: Idąc dalej

    * `Cloudflare`_, globalna platforma chmurowa;

    * `Dokumentacja pamięci podręcznej HTTP serwera Varnish`_;

    * `Specyfikacja ESI`_ i `zasoby programistów ESI`_;

    * `Model walidacji pamięci podręcznej HTTP`_;

    * `Pamięć podręczna HTTP (ang. HTTP cache) w Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`dokumentacji Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Dokumentacja pamięci podręcznej HTTP serwera Varnish`: https://varnish-cache.org/docs/index.html
.. _`Specyfikacja ESI`: https://www.w3.org/TR/esi-lang
.. _`zasoby programistów ESI`: https://www.akamai.com/us/en/support/esi.jsp
.. _`Model walidacji pamięci podręcznej HTTP`: https://symfony.com/doc/current/http_cache/validation.html
.. _`Pamięć podręczna HTTP (ang. HTTP cache) w Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
