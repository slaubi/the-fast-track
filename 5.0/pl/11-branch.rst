Zarządzanie gałęziami kodu
=============================

Istnieje wiele sposobów zarządzania zmianami kodu w projekcie. Tym niemniej, praca bezpośrednio na gałęzi głównej (ang. master) oraz bezpośrednie wdrożenia na środowisko produkcyjne z pominięciem testów prawdopodobnie nie są najlepszym pomysłem.

W testowaniu nie chodzi tylko o testy jednostkowe czy funkcjonalne, ale również o sprawdzanie zachowania aplikacji z danymi produkcyjnymi. Jeśli Ty lub Twoi `interesariusze`_ mogą przeglądać aplikację dokładnie tak, jak zostanie ona wdrożona dla użytkowników końcowych, staje się to ogromną zaletą i pozwala na bezpieczne wdrażanie. Jest to szczególnie ważne, jeśli ludzie nietechniczni mogą zatwierdzać nowe funkcje.

Dla uproszczenia i uniknięcia powtarzania się, w kolejnych krokach będziemy kontynuować całą pracę na gałęzi głównej (ang. master) w repozytorium Git, ale zobaczmy, jak to może działać lepiej.

Organizacja pracy z użyciem systemu Git
----------------------------------------

Jednym z możliwych sposobów pracy jest utworzenie jednej gałęzi dla każdej nowej funkcji lub poprawki błędu. Jest to proste i skuteczne.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Tworzenie gałęzi
------------------

.. index::
    single: Git;branch
    single: Git;checkout

Praca rozpoczyna się wraz z utworzeniem gałęzi Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

To polecenie tworzy gałąź ``sessions-in-redis`` z gałęzi ``master``. To działanie "rozwidla" kod i konfiguację infrastruktury.

Przechowywanie sesji w Redis
----------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Jak można się domyślić po nazwie gałęzi, chcemy przełączyć przechowywanie sesji z systemu plików na magazyn Redis.

Kroki niezbędne do urzeczywistnienia tej koncepcji są typowe:

#. Stwórz gałąź Git;

#. W razie potrzeby zaktualizuj konfigurację Symfony;

#. Napisz i/lub zaktualizuj kod, jeśli zajdzie taka potrzeba;

#. Zaktualizuj konfigurację PHP (dodaj rozszerzenie Redis w PHP);

#. Zaktualizuj infrastrukturę w Dockerze i SymfonyCloud (dodaj usługę Redis);

#. Przetestuj lokalnie;

#. Przetestuj zdalnie;

#. Połącz (ang. merge) bieżącą gałąź z gałęzią główną (ang. master);

#. Wdróż (ang. deploy) na produkcję;

#. Usuń gałąź.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Pozwolę Ci to przetestować lokalnie, poprzez przeglądanie strony internetowej. Ponieważ nie ma żadnych zmian wizualnych i nie korzystamy jeszcze z sesji, wszystko powinno działać tak jak wcześniej.

Wdrażanie gałęzi
-------------------

.. index::
    single: SymfonyCloud;Environment

Przed wdrożeniem (ang. deploy) na produkcję powinniśmy przetestować gałąź na tej samej infrastrukturze, co infrastruktura produkcyjna. Powinniśmy również sprawdzić, czy wszystko działa dobrze dla środowiska ``prod`` Symfony (lokalna strona internetowa korzystała ze środowiska ``dev`` Symfony).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Teraz stwórzmy *środowisko SymfonyCloud* oparte na *gałęzi Git* :

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Polecenie to tworzy nowe środowisko w następujący sposób:

* Gałąź dziedziczy kod i infrastrukturę po obecnej gałęzi Git (``sessions-in-redis``);

* Dane pochodzą ze środowiska głównego (zwanego też produkcyjnym) poprzez wykonanie spójnego obrazu danych z wszystkich usług, w tym plików (np. plików przesłanych przez użytkownika) i baz danych;

* Tworzony jest nowy dedykowany klaster do wdrożenia kodu, danych i infrastruktury.

Ponieważ wdrożenie odbywa się zgodnie z tymi samymi krokami co wdrożenie na produkcję, migracje do baz danych również będą wykonywane. Jest to świetny sposób na sprawdzenie, czy migracje działają z danymi produkcyjnymi.

Środowiska inne niż ``master`` są bardzo do niego podobne, z wyjątkiem kilku małych różnic: na przykład, wiadomości e-mail nie są domyślnie wysyłane.

.. index::
    single: Symfony CLI;open:remote

Po zakończeniu wdrożenia otwórz nową gałąź w przeglądarce:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Zauważ, że wszystkie polecenia SymfonyCloud działają na bieżącej gałęzi Git. Spowoduje to otwarcie adresu URL dla gałęzi ``sessions-in-redis`` - adres URL będzie wyglądał tak: ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Przetestuj stronę internetową na tym nowym środowisku, powinny być widoczne wszystkie dane, które zostały stworzone w środowisku głównym (ang. master).

Jeśli dodasz więcej konferencji w środowisku produkcyjnym ( gałąź ``master``), nie pojawią się one w środowisku ``sessions-in-redis`` i vice versa. Środowiska są niezależne i odizolowane.

Jeśli zmieni się kod w gałęzi głównej Git, zawsze można zrestartować gałąź i wdrożyć zaktualizowaną wersję, rozwiązując konflikty zarówno dla kodu, jak i infrastruktury.

.. index::
    single: Symfony CLI;env:sync

Można nawet zsynchronizować dane z gałęzi głównej (ang. master) z powrotem do środowiska ``sessions-in-redis``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Debugowanie wdrożeń produkcyjnych przed właściwym wdrożeniem
-----------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Domyślnie wszystkie środowiska SymfonyCloud używają tych samych ustawień, co środowisko ``master`` / ``prod`` (znane również jako środowisko ``prod`` Symfony). Pozwala to na przetestowanie aplikacji w rzeczywistych warunkach. Daje to poczucie rozwoju i testowania bezpośrednio na serwerach produkcyjnych, ale bez związanego z tym ryzyka. Przypomina mi to dawne dobre czasy, kiedy wdrożenia robiliśmy przez FTP.

.. index::
    single: Symfony CLI;env:debug

W przypadku wystąpienia problemu, możesz chcieć przełączyć się na środowisko ``dev`` Symfony:

.. code-block:: bash

    $ symfony env:debug

Po zakończeniu należy powrócić do ustawień produkcyjnych:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Nigdy** nie ustawiaj środowiska ``dev`` i nigdy nie włączaj Symfony Profiler na gałęzi ``master``; spowoduje to, że Twoja aplikacja będzie naprawdę powolna i otworzy wiele poważnych luk w zabezpieczeniach.

Testowanie wdrażania produkcji przed właściwym wdrożeniem
-------------------------------------------------------------

Dostęp do nowej wersji strony internetowej z danymi produkcyjnymi otwiera wiele możliwości: od testów regresji wizualnej po testy wydajnościowe. `Blackfire <https://blackfire.io>`_ jest idealnym narzędziem do tego zadania.

Przejdź do etapu poświęconego wydajności, aby dowiedzieć się więcej o tym, jak można użyć Blackfire do przetestowania kodu przed wdrożeniem.

Scalanie z produkcją
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Gdy jesteś zadowolony ze zmian w gałęzi, scal kod i infrastrukturę z powrotem do gałęzi głównej (ang. master) Git:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

I wdróż:

.. code-block:: bash

    $ symfony deploy

Podczas wdrażania, tylko kod i zmiany w infrastrukturze są przekazywane do SymfonyCloud; dane nie są w żaden sposób naruszone.

Sprzątanie
-----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

W końcu posprzątaj, usuwając gałąź Git i środowisko SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Idąc dalej

    * `Gałęzie w Gicie (ang. branching); <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_

    * `Redis docs <https://redis.io/documentation>`_.

.. _`interesariusze`: https://en.wikipedia.org/wiki/Project_stakeholder
