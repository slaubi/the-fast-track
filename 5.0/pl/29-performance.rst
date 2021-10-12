Zarządzanie wydajnością
==========================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Przedwczesna optymalizacja jest źródłem wszelkiego zła.

Ten cytat może już być Ci znany, ale lubię go w całości:

.. epigraph::

    Powinniśmy zapomnieć o małych usprawnieniach, powiedzmy w około 97% przypadków: przedwczesna optymalizacja jest źródłem wszelkiego zła. Nie powinniśmy jednak rezygnować z możliwości zwiększenia wydajności w tych krytycznych trzech procentach.

    --   Donald Knuth

Nawet niewielka poprawa wydajności może mieć znaczenie, zwłaszcza w przypadku sklepów internetowych. Teraz, gdy aplikacja księgi gości jest gotowa, zobaczmy, jak możemy sprawdzić jej wydajność.

Najlepszym sposobem na znalezienie optymalizacji wydajności jest użycie *profilera*. Najbardziej popularną obecnie opcją jest `Blackfire <https://blackfire.io>`_ (*Ważna informacja*: jestem również założycielem projektu Blackfire).

Przedstawienie Blackfire
------------------------

Blackfire składa się z kilku części:

* *Klient*, który uruchamia profilowanie (narzędzie Blackfire CLI lub rozszerzenie przeglądarki dla Google Chrome lub Firefox);

* *Agent*, który przygotowuje i zbiera dane przed wysłaniem ich do serwisu blackfire.io w celu ich wyświetlenia;

* Rozszerzenie PHP (*sonda*), które analizuje wykonanie kodu PHP.

Aby pracować z Blackfire, najpierw musisz `się zarejestrować <https://blackfire.io/signup>`_.

Zainstaluj Blackfire na swoim komputerze, uruchamiając następujący skrypt instalacyjny:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Instalator pobiera narzędzie Blackfire CLI, a następnie instaluje sondę PHP (bez włączania jej) na wszystkich dostępnych wersjach PHP.

Włącz sondę PHP dla naszego projektu:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Uruchom ponownie serwer WWW, aby PHP mógł załadować Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Aby powiązać profile z Twoim kontem, narzędzie Blackfire CLI po zainstalowaniu musi być skonfigurowane Twoimi danymi uwierzytelniającymi, które znajdziesz na górze `strony Settings/Credentials <https://blackfire.io/my/settings/credentials>`_. Wykonaj następujące polecenie podmieniając symbole zastępcze:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Aby uzyskać pełną instrukcję instalacji, postępuj zgodnie z `oficjalną szczegółową instrukcją instalacji <https://blackfire.io/docs/up-and-running/installation>`_. Jest przydatna podczas instalacji Blackfire na serwerze.

Instalowanie agenta Blackfire na Dockerze
-----------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Ostatnim krokiem jest dodanie usługi agenta Blackfire w stosie Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -20,3 +20,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Aby komunikować się z serwerem, musisz uzyskać swoje dane uwierzytelniające do **serwera** (te dane wskazują, gdzie chcesz przechowywać profile -- możesz utworzyć jeden na projekt); można je znaleźć na dole `strony Settings/Credentials <https://blackfire.io/my/settings/credentials>`_. Przechowuj je w lokalnym pliku ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Możesz już uruchomić nowy kontener:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Naprawianie niedziałającej instalacji Blackfire
-------------------------------------------------

Jeśli podczas profilowania pojawi się błąd, zwiększ poziom logowania Blackfire, aby uzyskać więcej informacji w logach:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Uruchom ponownie serwer WWW:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

A następnie śledź logi:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Wykonaj profilowanie ponownie i sprawdź wyjście logów.


Konfigurowanie Blackfire w środowisku produkcyjnym
---------------------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire jest domyślnie włączony dla wszystkich projektów SymfonyCloud.

Ustaw dane uwierzytelniające *serwera* jako zmienne środowiskowe:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

I włącz sondę PHP jak każde inne rozszerzenie PHP:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - amqp
             - redis

Konfigurowanie serwera Varnish dla Blackfire
--------------------------------------------

.. index::
    single: SymfonyCloud;Varnish

Zanim będziesz w stanie wykonać wdrożenie potrzebne do rozpoczęcia profilowania, potrzebujesz sposobu na ominięcie pamięci podręcznej Varnish HTTP. W innym przypadku, Blackfire nigdy nie odpyta aplikacji PHP. Będziesz autoryzować tylko prośby o profilowanie pochodzące z twojej lokalnej maszyny.

Znajdź swój bieżący adres IP:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

I użyj go, aby skonfigurować Varnish:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

Teraz możesz wdrożyć.

Profilowanie stron internetowych
--------------------------------

.. index::
    single: Profiling;Web Pages

Możesz profilować tradycyjne strony internetowe używając Firefoksa lub Google Chrome poprzez ich `dedykowane rozszerzenia <https://blackfire.io/docs/integrations/browsers/index>`_.

Na lokalnym komputerze, podczas profilowania, nie zapomnij wyłączyć pamięci podręcznej HTTP w ``public/index.php``: jeśli tego nie zrobisz, będziesz profilować warstwę pamięci podręcznej HTTP Symfony zamiast własnego kodu:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/public/index.php
    +++ b/public/index.php
    @@ -24,7 +24,7 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? $_ENV['TRUSTED_HOSTS'] ?? false
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);

     if ('dev' === $kernel->getEnvironment()) {
    -    $kernel = new HttpCache($kernel);
    +//    $kernel = new HttpCache($kernel);
     }

     $request = Request::createFromGlobals();

Aby uzyskać lepszy obraz wydajności Twojej aplikacji w środowisku produkcyjnym, należy je również profilować. Domyślnie, twoje lokalne środowisko korzysta ze środowiska deweloperskiego, które dodaje znaczny narzut (głównie w celu zebrania danych dla paska narzędzi do debugowania sieci i profilera Symfony).

.. index::
    single: Symfony CLI;server:prod

Przełączenie maszyny lokalnej do środowiska produkcyjnego można wykonać poprzez zmianę zmiennej środowiskowej ``APP_ENV`` w pliku ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Albo możesz użyć polecenia ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Nie zapomnij przełączyć go z powrotem na środowisko deweloperskie po zakończeniu sesji profilowania:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Profilowanie zasobów API
-------------------------

.. index::
    single: Profiling;API

Profilowanie API lub SPA jest wygodniejsze z poziomu linii komend, za pomocą zainstalowanego wcześniej narzędzia Blackfire CLI:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Polecenie ``blackfire curl`` akceptuje dokładnie takie same argumenty i opcje jak `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Porównanie wydajności
-----------------------

W etapie poświęconym pamięci podręcznej dodaliśmy warstwę pamięci podręcznej, aby poprawić wydajność naszego kodu, ale nie sprawdziliśmy ani nie zmierzyliśmy wpływu zmiany na wydajność. Ponieważ źle nam idzie zgadywanie, co będzie szybkie, a co powolne, możesz znaleźć się w sytuacji, w której niektóre usprawnienia spowalniają działanie Twojej aplikacji.

Za każdym razem należy zmierzyć wpływ wszelkich optymalizacji, które robisz za pomocą profilera. Blackfire ułatwia wizualne ich porównanie dzięki `narzędziu porównywania <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Tworzenie czarnoskrzynkowych testów funkcjonalnych (ang. black box)
--------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Dowiedzieliśmy się już, jak pisać testy funkcjonalne z Symfony. Blackfire może być używany do pisania scenariuszy przeglądania, które mogą być uruchamiane na żądanie przez `odtwarzacz Blackfire <https://blackfire.io/player>`_. Napiszmy scenariusz, który przesyła nowy komentarz i zatwierdza go poprzez link e-mailowy w środowisku deweloperskim oraz przez konto administracyjne w środowisku produkcyjnym.

Utwórz plik ``.blackfire.yaml`` o następującej zawartości:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Pobierz odtwarzacz Blackfire, aby móc uruchomić scenariusz lokalnie:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Uruchomienie tego scenariusza w środowisku deweloperskim:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Albo w środowisku produkcyjnym:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Scenariusze Blackfire mogą również wyzwalać profilowanie dla każdego żądania i uruchamiać testy wydajności poprzez dodanie flagi ``--blackfire``.

Automatyzowanie kontroli wydajności
------------------------------------

Zarządzanie wydajnością polega nie tylko na poprawie wydajności istniejącego kodu, ale również na sprawdzeniu, czy dotychczasowe działania wydajności nie zmniejszyły.

Scenariusz napisany w poprzedniej sekcji może być uruchamiany automatycznie w trybie ciągłej integracji (ang. continuous integration) lub w środowisku produkcyjnym w regularnych odstępach.

SymfonyCloud pozwala również na `uruchamianie scenariuszy <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_, gdy tworzysz nową gałąź lub wdrażasz w środowisku produkcyjnym, aby automatycznie sprawdzić wydajność nowego kodu.

.. sidebar:: Idąc dalej

    * `Książka Blackfire: wydajność kodu PHP wyjaśniona <https://blackfire.io/book>`_;

    * `Samouczek SymfonyCasts Blackfire <https://symfonycasts.com/screencast/blackfire>`_.
