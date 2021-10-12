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

Na naszej lokalnej maszynie zdecydowaliśmy się użyć Dockera do zarządzania usługami. Utwórz plik ``docker-compose.yaml`` i dodaj PostgreSQL jako usługę:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

Ta operacja zainstaluje serwer PostgreSQL i skonfiguruje niektóre zmienne środowiskowe, które kontrolują nazwę bazy danych i dane uwierzytelniające. Wartości nie mają większego znaczenia.

Udostępniamy również port PostgreSQL (``5432``) kontenera do lokalnego hosta. To pomoże nam uzyskać dostęp do bazy danych z naszej maszyny.

.. note::

    Rozszerzenie ``pdo_pgsql`` powinno być zainstalowane po tym, jak konfigurowaliśmy PHP w poprzednim kroku.

Uruchamianie Docker Compose
---------------------------

Uruchom Docker Compose w tle ( ``-d`` ):

.. code-block:: bash

    $ docker-compose up -d

Poczekaj chwilę, aż baza danych się uruchomi i sprawdź, czy wszystko działa prawidłowo:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Jeśli nie ma uruchomionych kontenerów lub jeśli kolumna ``State`` nie ma wartości ``Up``, sprawdź logi Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Dostęp do lokalnej bazy danych
-------------------------------

Korzystanie z narzędzia ``psql`` w linii poleceń może okazać się przydatne od czasu do czasu, ale musisz wtedy znać dane uwierzytelniające i nazwę bazy danych a także, co jest mniej oczywiste, lokalny port, na którym działa baza danych. Docker wybiera losowy port, dzięki czemu możesz pracować nad więcej niż jednym projektem korzystającym z serwera PostgreSQL w tym samym czasie (port lokalny jest częścią danych wyjściowych ``docker-compose ps``).

Jeśli uruchomisz ``psql`` za pomocą Symfony CLI nie musisz o niczym pamiętać.

Symfony CLI automatycznie wykrywa usługi Docker uruchomione dla projektu i udostępnia zmienne środowiskowe, których ``psql`` potrzebuje do połączenia z bazą danych.

.. index::
    single: Symfony CLI;run psql

Dzięki tym zasadom dostęp do bazy danych poprzez ``symfony run``  jest znacznie łatwiejszy:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    Jeśli nie masz binarnej wersji ``psql`` na swoim komputerze, możesz uruchomić go za pośrednictwem polecenia ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Zrzucanie i przywracanie bazy danych
------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Skorzystaj z polecenia ``pg_dump`` w celu wykonania zrzutu bazy danych:

.. code-block:: bash
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

I przywróć bazę:

.. code-block:: bash
    :class: ignore

    $ symfony run psql < dump.sql

.. warning::

    Nigdy nie wykonuj polecenia ``docker-compose down``, jeżeli nie chcesz utracić danych w kontenerach, chyba że wcześniej wykonasz kopię tych danych.

Dodawanie PostgreSQL do SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

W przypadku infrastruktury produkcyjnej na SymfonyCloud, dodanie usługi takiej jak PostgreSQL powinno być wykonane w aktualnie pustym pliku ``.symfony/services.yaml``:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Usługa ``db`` jest bazą danych PostgreSQL (w takiej wersji, jaka jest w Dockerze), którą chcemy udostępnić w małym kontenerze z dyskiem o powierzchni 1GB.

Musimy również "połączyć" bazę danych z kontenerem aplikacji, który jest opisany w ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Usługa ``db``  typu ``postgresql`` jest określona jako ``database`` w kontenerze aplikacyjnym.

Ostatnim krokiem jest dodanie rozszerzenia ``pdo_pgsql`` do środowiska uruchomieniowego PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Poniżej znajduje się pełna lista zmian (ang. diff) dla ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

Zatwierdź (ang. commit) te zmiany, a następnie wdróż (ang. deploy) je do SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Dostęp do bazy danych w SymfonyCloud
-------------------------------------

PostgreSQL działa teraz zarówno lokalnie poprzez Dockera, jak i na produkcji w SymfonyCloud.

Jak właśnie widzieliśmy, uruchamianie ``symfony run psql`` automatycznie łączy się z bazą danych hostowaną przez Dockera dzięki zmiennym środowiskowym udostępnionym przez ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Jeśli chcesz się połączyć z PostgreSQL hostowanym w kontenerach produkcyjnych, możesz otworzyć tunel SSH pomiędzy lokalnym komputerem a infrastrukturą SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

Domyślnie usługi SymfonyCloud nie są udostępniane jako zmienne środowiskowe na lokalnym komputerze. Musisz to zrobić samodzielnie, korzystając z flagi ``--expose-env-vars``. Dlaczego? Podłączenie do produkcyjnej bazy danych jest niebezpieczną operacją. Możesz w ten sposób namieszać z *prawdziwymi* danymi. Wymaganie flagi to sposób potwierdzenia, że to jest *na pewno* to, co chcesz zrobić.

Połącz się teraz ze zdalną bazą danych PostgreSQL korzystając z ``symfony run psql``, jak poprzednio:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Kiedy skończysz, nie zapomnij zamknąć tunelu:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Aby uruchomić niektóre zapytania SQL w produkcyjnej bazie danych zamiast korzystać z powłoki, możesz wykorzystać polecenie ``symfony sql``.

Udostępnianie zmiennych środowiskowych
----------------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose i SymfonyCloud dobrze współpracują z Symfony dzięki zmiennym środowiskowym.

Sprawdź wszystkie zmienne środowiskowe udostępnione przez ``symfony`` poprzez wykonanie polecenia ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Zmienne środowiskowe ``PG*`` są odczytywane przez narzędzie ``psql``. A co z innymi?

Kiedy tunel jest zestawiony z SymfonyCloud z ustawioną flagą ``--expose-env-vars``, polecenie ``var:export`` zwraca również zdalne zmienne środowiskowe:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

Opisywanie infrastruktury
-------------------------

Być może jeszcze nie zdawałeś sobie z tego sprawy, ale posiadanie infrastruktury przechowywanej w plikach wraz z kodem bardzo pomaga. Docker i SymfonyCloud używają plików konfiguracyjnych do opisania infrastruktury projektu. Gdy nowa funkcja wymaga dodatkowej usługi, zmiany kodu i zmiany infrastruktury są częścią tej samej poprawki.

.. sidebar:: Idąc dalej

    * `Usługi SymfonyCloud <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `Tunel SymfonyCloud <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Dokumentacja PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `Polecenia docker-compose <https://docs.docker.com/compose/reference/>`_
