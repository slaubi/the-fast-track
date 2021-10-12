Odkrywanie wewnętrznych mechanizmów Symfony
=============================================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

Od dłuższego czasu używamy Symfony do stworzenia potężnej aplikacji i większość kodu wykonywanego przez aplikację pochodzi nie od nas, ale z Symfony. Kilkaset linii kodu kontra tysiące linii kodu.

Lubię rozumieć, jak rzeczy działają od środka. Zawsze fascynowały mnie narzędzia, które pomagają zrozumieć, jak coś działa. Pierwsze użycie debuggera lub wykorzystanie po raz pierwszy funkcji ``ptrace`` to dla mnie magiczne wspomnienia.

Chcesz lepiej zrozumieć, jak działa Symfony? Czas zgłębić mechanizmy, jakich Symfony używa, aby Twoja aplikacja robiła to, co robi. Zamiast opisywać z teoretycznej perspektywy, jak Symfony radzi sobie z żądaniem HTTP – co byłoby dość nudne – wykorzystamy narzędzie Blackfire, aby uzyskać pewne wizualne reprezentacje i skorzystać z nich do przedstawienia bardziej zaawansowanych problemów.

Zrozumienie wewnętrznych mechanizmów Symfony przy użyciu Blackfire
---------------------------------------------------------------------

Już wiesz, że wszystkie żądania HTTP są obsługiwane przez jeden punkt wejścia: plik ``public/index.php``. Ale co dalej? Jak kontrolery są wywoływane?

Sprofilujmy produkcyjną, angielską wersję strony głównej przy użyciu Blackfire poprzez wtyczkę Blackfire do przeglądarki:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

Albo bezpośrednio w wierszu poleceń:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Po przejściu do widoku "Oś czasu" w profilu, znajdziesz widok podobny do poniższego:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

Najedź na kolorowe paski na osi czasu, aby uzyskać więcej informacji na temat każdego wywołania; poznasz lepiej działanie Symfony:

* Głównym punktem wejścia jest ``public/index.php``;

* Metoda ``Kernel::handle()`` obsługuje żądanie;

* Wywołuje ``HttpKernel``, który emituje pewne zdarzenia;

* Pierwszym zdarzeniem jest ``RequestEvent``;

* Metoda ``ControllerResolver::getController()`` jest wywoływana aby określić, który kontroler powinien być wywoływany dla przychodzącego adresu URL;

* Metoda ``ControllerResolver::getArguments()`` jest wywoływana aby określić, które argumenty mają być przekazane do kontrolera (wywoływany jest konwerter parametrów);

* W następnie wywoływanej metodzie ``ConferenceController::index()`` wykonywana jest większość naszego kodu;

* Metoda ``ConferenceRepository::findAll()`` pobiera wszystkie konferencje z bazy danych (zwróć uwagę na połączenie z bazą danych za pośrednictwem ``PDO::__construct()``);

* Metoda ``Twig\Environment::render()`` renderuje szablon;

* Zdarzenia ``ResponseEvent`` oraz ``FinishRequestEvent`` są wywoływane, ale wygląda na to, że żaden nasłuchiwacz zdarzeń (ang. event listener) nie jest zarejestrowany, ponieważ zdarzenia wykonują się naprawdę szybko.

Oś czasu to świetny sposób na zrozumienie, jak działa kod; jest to bardzo przydatne, kiedy otrzymujesz projekt rozwijany przez kogoś innego.

Teraz sprofiluj tę samą stronę z maszyny lokalnej w środowisku deweloperskim:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Otwórz profil. Powinno nastąpić przekierowanie do widoku grafu połączeń, a ponieważ żądanie było naprawdę szybkie, oś czasu będzie prawie pusta:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Rozumiesz, co się dzieje? Pamięć podręczna HTTP (ang. HTTP cache) jest włączona i przez to profilujemy warstwę pamięci podręcznej Symfony. Ponieważ strona znajduje się w pamięci podręcznej, ``HttpCache\Store::restoreResponse()`` otrzymuje odpowiedź HTTP z pamięci podręcznej i kontroler nigdy nie jest wywoływany.

Wyłącz warstwę pamięci podręcznej w ``public/index.php`` tak jak w poprzednim etapie i spróbuj ponownie. Od razu widać, że profil wygląda zupełnie inaczej:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Główne różnice są następujące:

* Zdarzenie ``TerminateEvent``, które nie było widoczne w środowisku produkcyjnym, zajmuje duży procent czasu wykonania; możemy dostrzec, że jest to zdarzenie odpowiedzialne za przechowywanie danych profilera Symfony zebranych podczas żądania;

* W wywołaniu ``ConferenceController::index()``, zwróć uwagę na ``SubRequestHandler::handle()``, metodę, która renderuje ESI (dlatego mamy dwa wywołania ``Profiler::saveProfile()``, jedno dla głównego żądania i jedno dla ESI).

Przejdź do osi czasu, aby dowiedzieć się więcej; przejdź do widoku grafu połączeń, aby zobaczyć różne reprezentacje tych samych danych.

Jak właśnie odkryliśmy, kod wykonywany w środowisku deweloperskim i produkcyjnym jest zupełnie inny. Środowisko programistyczne jest wolniejsze, ponieważ profiler Symfony próbuje zebrać wiele danych, aby ułatwić debugowanie. Dlatego też należy zawsze stosować profilowanie w środowisku produkcyjnym, nawet lokalnie.

W ramach ciekawych eksperymentów: sprofiluj stronę błędu, stronę ``/`` (która jest przekierowaniem) lub zasób API. Każdy profil powie Ci trochę więcej o tym, jak działa Symfony, jakie klasy/metody są wywoływane, co ma większy koszt wykonania a co niższy.

Używanie dodatku Blackfire Debug
---------------------------------

.. index::
    single: Blackfire;Debug Addon

Domyślnie Blackfire usuwa wszystkie wywołania metod, które nie są wystarczająco znaczące, aby uniknąć ładowania zbyt dużej ilości danych i pokazywania dużych wykresów. W przypadku używania Blackfire jako narzędzia do debugowania, lepiej jest zachować wszystkie wywołania używając dodatku debug.

W wierszu poleceń, użyj flagi ``--debug``:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

W środowisku produkcyjnym, zobaczysz na przykład ładowanie pliku o nazwie ``.env.local.php``:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

Skąd to się wzięło? SymfonyCloud wykonuje pewne optymalizacje podczas wdrażania aplikacji Symfony, np. optymalizując autoloader Composer (``--optimize-autoloader --apcu-autoloader --classmap-authoritative``) oraz zmienne środowiskowe zdefiniowane w pliku ``.env`` (aby uniknąć przetwarzania pliku dla każdego żądania) poprzez wygenerowanie pliku ``.env.local.php``:

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire jest bardzo potężnym narzędziem, które pomaga zrozumieć ,w jaki sposób kod jest wykonywany przez PHP. Poprawa wydajności jest tylko jednym ze sposobów wykorzystania profilera.
