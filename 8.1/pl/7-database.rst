Konfigurowanie bazy danych
==========================

.. index::
    single: Database

Strona internetowa księgi gości konferencji dotyczy zbierania opinii podczas konferencji. Komentarze uczestników konferencji muszą być utrwalone w bazie danych.

Komentarz najlepiej opisuje ustalona struktura danych: autor, jego e-mail, tekst opinii i opcjonalne zdjęcie. Jest to rodzaj danych, które mogą być wygodnie przechowywane w tradycyjnym silniku relacyjnej bazy danych.

PostgreSQL to silnik bazodanowy, którego będziemy używać.

Dodawanie PostgreSQL do Docker Compose
--------------------------------------

.. index::
    single: Docker;PostgreSQL

Na naszej lokalnej maszynie zdecydowaliśmy się użyć Dockera do zarządzania usługami. Wygenerowany plik ``compose.yaml`` już zawiera PostgreSQL jako usługę:

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

Ta operacja zainstaluje serwer PostgreSQL i skonfiguruje niektóre zmienne środowiskowe, które kontrolują nazwę bazy danych i dane uwierzytelniające. Wartości nie mają większego znaczenia.

Udostępniamy również port PostgreSQL (``5432``) kontenera do lokalnego hosta. To pomoże nam uzyskać dostęp do bazy danych z naszej maszyny:

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    Rozszerzenie ``pdo_pgsql`` powinno być zainstalowane po tym, jak konfigurowaliśmy PHP w poprzednim kroku.

Uruchamianie Docker Compose
---------------------------

Uruchom Docker Compose w tle ( ``-d`` ):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

Poczekaj chwilę, aż baza danych się uruchomi i sprawdź, czy wszystko działa prawidłowo:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Jeśli nie ma uruchomionych kontenerów lub jeśli kolumna ``State`` nie ma wartości ``Up``, sprawdź logi Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

Dostęp do lokalnej bazy danych
-------------------------------

Korzystanie z narzędzia ``psql`` w linii poleceń może okazać się przydatne od czasu do czasu, ale musisz wtedy znać dane uwierzytelniające i nazwę bazy danych a także, co jest mniej oczywiste, lokalny port, na którym działa baza danych. Docker wybiera losowy port, dzięki czemu możesz pracować nad więcej niż jednym projektem korzystającym z serwera PostgreSQL w tym samym czasie (port lokalny jest częścią danych wyjściowych ``docker-compose ps``).

Jeśli uruchomisz ``psql`` za pomocą Symfony CLI nie musisz o niczym pamiętać.

Symfony CLI automatycznie wykrywa usługi Docker uruchomione dla projektu i udostępnia zmienne środowiskowe, których ``psql`` potrzebuje do połączenia z bazą danych.

.. index::
    single: Symfony CLI;run psql

Dzięki tym zasadom dostęp do bazy danych poprzez ``symfony run``  jest znacznie łatwiejszy:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Jeśli nie masz binarnej wersji ``psql`` na swoim komputerze, możesz uruchomić go za pośrednictwem polecenia ``docker compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

Zrzucanie i przywracanie bazy danych
------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Skorzystaj z polecenia ``pg_dump`` w celu wykonania zrzutu bazy danych:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

I przywróć bazę:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Dodawanie PostgreSQL do Upsun
-----------------------------------

.. index::
    single: Upsun;PostgreSQL

W przypadku infrastruktury produkcyjnej na Upsun, dodanie usługi takiej jak PostgreSQL powinno być wykonane w pliku ``.upsun/config.yaml``, który został utworzony za pomocą przepisu z pakietu ``webapp``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

Usługa ``database`` jest bazą danych PostgreSQL (w takiej wersji, jaka jest w Dockerze). Upsun automatycznie przydziela dysk podczas pierwszego wdrożenia; w razie potrzeby dostosuj go później za pomocą ``symfony cloud:resources:set``.

Musimy również "połączyć" bazę danych z kontenerem aplikacji, który jest opisany w ``.upsun/config.yaml``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Usługa ``database``  typu ``postgresql`` jest określona jako ``database`` w kontenerze aplikacyjnym.

Sprawdź czy rozszerzenie ``pdo_pgsql`` jest już zainstalowane dla środowiska uruchomieniowego PHP:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Dostęp do bazy danych w Upsun
------------------------------------

PostgreSQL działa teraz zarówno lokalnie poprzez Dockera, jak i na produkcji w Upsun.

Jak właśnie widzieliśmy, uruchamianie ``symfony run psql`` automatycznie łączy się z bazą danych hostowaną przez Dockera dzięki zmiennym środowiskowym udostępnionym przez ``symfony run``.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Jeśli chcesz się połączyć z PostgreSQL hostowanym w kontenerach produkcyjnych, możesz otworzyć tunel SSH pomiędzy lokalnym komputerem a infrastrukturą Upsun:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Domyślnie usługi Upsun nie są udostępniane jako zmienne środowiskowe na lokalnym komputerze. Musisz to zrobić samodzielnie, korzystając z flagi ``--expose-env-vars``. Dlaczego? Podłączenie do produkcyjnej bazy danych jest niebezpieczną operacją. Możesz w ten sposób namieszać z *prawdziwymi* danymi.

Połącz się teraz ze zdalną bazą danych PostgreSQL korzystając z ``symfony run psql``, jak poprzednio:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Kiedy skończysz, nie zapomnij zamknąć tunelu:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Aby uruchomić niektóre zapytania SQL w produkcyjnej bazie danych zamiast korzystać z powłoki, możesz wykorzystać polecenie ``symfony sql``.

Udostępnianie zmiennych środowiskowych
----------------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

Docker Compose i Upsun dobrze współpracują z Symfony dzięki zmiennym środowiskowym.

Sprawdź wszystkie zmienne środowiskowe udostępnione przez ``symfony`` poprzez wykonanie polecenia ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Zmienne środowiskowe ``PG*`` są odczytywane przez narzędzie ``psql``. A co z innymi?

Kiedy tunel jest zestawiony z Upsun z ``var:expose-from-tunnel``, polecenie ``var:export`` zwraca również zdalne zmienne środowiskowe:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Opisywanie infrastruktury
-------------------------

Być może jeszcze nie zdawałeś sobie z tego sprawy, ale posiadanie infrastruktury przechowywanej w plikach wraz z kodem bardzo pomaga. Docker i Upsun używają plików konfiguracyjnych do opisania infrastruktury projektu. Gdy nowa funkcja wymaga dodatkowej usługi, zmiany kodu i zmiany infrastruktury są częścią tej samej poprawki.

.. sidebar:: Idąc dalej

    * `Usługi Upsun`_;

    * `Tunel Upsun`_;

    * `Dokumentacja PostgreSQL`_;

    * `Polecenia Docker Compose`_

.. _`Usługi Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Tunel Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Dokumentacja PostgreSQL`: https://www.postgresql.org/docs/
.. _`Polecenia Docker Compose`: https://docs.docker.com/reference/cli/docker/compose/
