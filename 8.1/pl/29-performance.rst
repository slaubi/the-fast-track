Zarządzanie wydajnością
==========================

.. index::
    single: Blackfire
    single: Profiler

Przedwczesna optymalizacja jest źródłem wszelkiego zła.

Ten cytat może już być Ci znany, ale lubię go w całości:

Powinniśmy zapomnieć o małych usprawnieniach, powiedzmy w około 97% przypadków: przedwczesna optymalizacja jest źródłem wszelkiego zła. Nie powinniśmy jednak rezygnować z możliwości zwiększenia wydajności w tych krytycznych trzech procentach.

--   Donald Knuth

Nawet niewielka poprawa wydajności może mieć znaczenie, zwłaszcza w przypadku sklepów internetowych. Teraz, gdy aplikacja księgi gości jest gotowa, zobaczmy, jak możemy sprawdzić jej wydajność.

Najlepszym sposobem na znalezienie optymalizacji wydajności jest użycie *profilera*. Najbardziej popularną obecnie opcją jest `Blackfire`_ (*Ważna informacja*: jestem również założycielem projektu Blackfire).

Przedstawienie Blackfire
------------------------

Blackfire składa się z kilku części:

* *Klient*, który uruchamia profilowanie (narzędzie Blackfire CLI lub rozszerzenie przeglądarki dla Google Chrome lub Firefox);

* *Agent*, który przygotowuje i zbiera dane przed wysłaniem ich do serwisu blackfire.io w celu ich wyświetlenia;

* Rozszerzenie PHP (*sonda*), które analizuje wykonanie kodu PHP.

Aby pracować z Blackfire, najpierw musisz `się zarejestrować`_.

Zainstaluj Blackfire na swoim komputerze, uruchamiając następujący skrypt instalacyjny:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

Instalator pobiera i instaluje narzędzie Blackfire CLI.

Po zakończeniu, zainstaluj sondę dla wszystkich dostępnych wersji PHP:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

Włącz też sondę PHP dla naszego projektu:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Uruchom ponownie serwer WWW, aby PHP mógł załadować Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Aby powiązać profile z Twoim kontem, narzędzie Blackfire CLI po zainstalowaniu musi być skonfigurowane Twoimi danymi uwierzytelniającymi, które znajdziesz na górze `strony Settings/Credentials`_. Wykonaj następujące polecenie podmieniając symbole zastępcze:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

Instalowanie agenta Blackfire na Dockerze
-----------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Usługa agenta Blackfire została już skonfigurowana w stosie Docker Compose:

.. code-block:: yaml
    :caption: compose.override.yaml
    :class: ignore

    ###> blackfireio/blackfire-symfony-meta ###
    blackfire:
        image: blackfire/blackfire:2
        # uncomment to store Blackfire credentials in a local .env.local file
        #env_file: .env.local
        environment:
        BLACKFIRE_LOG_LEVEL: 4
        ports: [8307]
    ###< blackfireio/blackfire-symfony-meta ###

Aby komunikować się z serwerem, musisz uzyskać swoje dane uwierzytelniające do **serwera** (te dane wskazują, gdzie chcesz przechowywać profile -- możesz utworzyć jeden na projekt); można je znaleźć na dole `strony Settings/Credentials`_. Przechowuj je pośród poufnych danych:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Możesz już uruchomić nowy kontener:

.. code-block:: terminal
    :class: ignore

    $ docker compose stop
    $ docker compose up -d --remove-orphans

Naprawianie niedziałającej instalacji Blackfire
-------------------------------------------------

Jeśli podczas profilowania pojawi się błąd, zwiększ poziom logowania Blackfire, aby uzyskać więcej informacji w logach:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/php.ini
    +++ w/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Uruchom ponownie serwer WWW:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

A następnie śledź logi:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Wykonaj profilowanie ponownie i sprawdź wyjście logów.

Konfigurowanie Blackfire w środowisku produkcyjnym
---------------------------------------------------

.. index::
    single: Upsun;Blackfire

Blackfire jest domyślnie włączony dla wszystkich projektów Upsun.

Ustaw dane uwierzytelniające *serwera* jako poufne dane **produkcyjne**:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Sonda PHP jest już włączona, jak każde inne potrzebne rozszerzenie PHP:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 9
    :class: ignore

    runtime:
        extensions:
            - apcu
            - blackfire
            - ctype
            - iconv
            - mbstring
            - pdo_pgsql
            - sodium
            - xsl

Konfigurowanie serwera Varnish dla Blackfire
--------------------------------------------

.. index::
    single: Upsun;Varnish

Zanim będziesz w stanie wykonać wdrożenie potrzebne do rozpoczęcia profilowania, potrzebujesz sposobu na ominięcie pamięci podręcznej Varnish HTTP. W innym przypadku, Blackfire nigdy nie odpyta aplikacji PHP. Będziesz autoryzować tylko prośby o profilowanie pochodzące z twojej lokalnej maszyny.

Znajdź swój bieżący adres IP:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

I użyj go, aby skonfigurować Varnish:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "192.168.0.1";
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

Możesz profilować tradycyjne strony internetowe używając Firefoksa lub Google Chrome poprzez ich `dedykowane rozszerzenia`_.

Na lokalnym komputerze, podczas profilowania, nie zapomnij wyłączyć pamięci podręcznej HTTP w ``config/packages/framework.yaml``: jeśli tego nie zrobisz, będziesz profilować warstwę pamięci podręcznej HTTP Symfony zamiast własnego kodu.

Aby uzyskać lepszy obraz wydajności Twojej aplikacji w środowisku produkcyjnym, należy je również profilować. Domyślnie, twoje lokalne środowisko korzysta ze środowiska deweloperskiego, które dodaje znaczny narzut (głównie w celu zebrania danych dla paska narzędzi do debugowania sieci i profilera Symfony).

.. note::

    Ponieważ będziemy profilować środowisko produkcyjne, nie ma nic do zmiany w konfiguracji, ponieważ w poprzednim rozdziale włączyliśmy warstwę pamięci podręcznej HTTP Symfony tylko dla środowiska produkcyjnego.

.. index::
    single: Symfony CLI;server:prod

Przełączenie maszyny lokalnej do środowiska produkcyjnego można wykonać poprzez zmianę zmiennej środowiskowej ``APP_ENV`` w pliku ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Albo możesz użyć polecenia ``server:prod``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

Nie zapomnij przełączyć go z powrotem na środowisko deweloperskie po zakończeniu sesji profilowania:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

Profilowanie zasobów API
-------------------------

.. index::
    single: Profiling;API

Profilowanie API lub SPA jest wygodniejsze z poziomu linii komend, za pomocą zainstalowanego wcześniej narzędzia Blackfire CLI:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Polecenie ``blackfire curl`` akceptuje dokładnie takie same argumenty i opcje jak `cURL`_.

Porównanie wydajności
-----------------------

W etapie poświęconym pamięci podręcznej dodaliśmy warstwę pamięci podręcznej, aby poprawić wydajność naszego kodu, ale nie sprawdziliśmy ani nie zmierzyliśmy wpływu zmiany na wydajność. Ponieważ źle nam idzie zgadywanie, co będzie szybkie, a co powolne, możesz znaleźć się w sytuacji, w której niektóre usprawnienia spowalniają działanie Twojej aplikacji.

Za każdym razem należy zmierzyć wpływ wszelkich optymalizacji, które robisz za pomocą profilera. Blackfire ułatwia wizualne ich porównanie dzięki `narzędziu porównywania`_.

Tworzenie czarnoskrzynkowych testów funkcjonalnych (ang. black box)
--------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Dowiedzieliśmy się już, jak pisać testy funkcjonalne z Symfony. Blackfire może być używany do pisania scenariuszy przeglądania, które mogą być uruchamiane na żądanie przez `odtwarzacz Blackfire`_. Napiszmy scenariusz, który przesyła nowy komentarz i zatwierdza go poprzez link e-mailowy w środowisku deweloperskim oraz przez konto administracyjne w środowisku produkcyjnym.

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
                param comment[author] 'Fabien'
                param comment[email] 'me@example.com'
                param comment[text] 'Such a good conference!'
                param comment[photo] file(fake('simple_image', '/tmp', 400, 300, 'png', true, true), 'placeholder-image.jpg')
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
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin')
                    expect status_code() == 302
                follow
                click link("Comments")
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

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Uruchomienie tego scenariusza w środowisku deweloperskim:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

Albo w środowisku produkcyjnym:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

Scenariusze Blackfire mogą również wyzwalać profilowanie dla każdego żądania i uruchamiać testy wydajności poprzez dodanie flagi ``--blackfire``.

Automatyzowanie kontroli wydajności
------------------------------------

Zarządzanie wydajnością polega nie tylko na poprawie wydajności istniejącego kodu, ale również na sprawdzeniu, czy dotychczasowe działania wydajności nie zmniejszyły.

Scenariusz napisany w poprzedniej sekcji może być uruchamiany automatycznie w trybie ciągłej integracji (ang. continuous integration) lub w środowisku produkcyjnym w regularnych odstępach.

Upsun pozwala również na `uruchamianie scenariuszy`_, gdy tworzysz nową gałąź lub wdrażasz w środowisku produkcyjnym, aby automatycznie sprawdzić wydajność nowego kodu.

.. sidebar:: Idąc dalej

    * `Książka Blackfire: wydajność kodu PHP wyjaśniona`_;

    * `Samouczek SymfonyCasts Blackfire`_.

.. _`Blackfire`: https://blackfire.io
.. _`się zarejestrować`: https://blackfire.io/signup
.. _`strony Settings/Credentials`: https://blackfire.io/my/settings/credentials
.. _`dedykowane rozszerzenia`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`narzędziu porównywania`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`odtwarzacz Blackfire`: https://blackfire.io/player
.. _`uruchamianie scenariuszy`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`Książka Blackfire: wydajność kodu PHP wyjaśniona`: https://blackfire.io/book
.. _`Samouczek SymfonyCasts Blackfire`: https://symfonycasts.com/screencast/blackfire
