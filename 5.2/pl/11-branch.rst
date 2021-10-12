Zarządzanie gałęziami kodu
=============================

Istnieje wiele sposobów zarządzania zmianami kodu w projekcie. Tym niemniej, praca bezpośrednio na gałęzi głównej (ang. master) oraz bezpośrednie wdrożenia na środowisko produkcyjne z pominięciem testów prawdopodobnie nie są najlepszym pomysłem.

W testowaniu nie chodzi tylko o testy jednostkowe czy funkcjonalne, ale również o sprawdzanie zachowania aplikacji z danymi produkcyjnymi. Jeśli Ty lub Twoi `interesariusze`_ mogą przeglądać aplikację dokładnie tak, jak zostanie ona wdrożona dla użytkowników końcowych, staje się to ogromną zaletą i pozwala na bezpieczne wdrażanie. Jest to szczególnie ważne, jeśli ludzie nietechniczni mogą zatwierdzać nowe funkcje.

Dla uproszczenia i uniknięcia powtarzania się, w kolejnych krokach będziemy kontynuować całą pracę na gałęzi głównej (ang. master) w repozytorium Git, ale zobaczmy, jak to może działać lepiej.

Organizacja pracy z użyciem systemu Git
----------------------------------------

Jednym z możliwych sposobów pracy jest utworzenie jednej gałęzi dla każdej nowej funkcji lub poprawki błędu. Jest to proste i skuteczne.

Tworzenie gałęzi
------------------

.. index::
    single: Git;branch
    single: Git;checkout

Praca rozpoczyna się wraz z utworzeniem gałęzi Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: bash

    $ git checkout -b sessions-in-db

To polecenie stworzy gałąź ``sessions-in-db`` z gałęzi ``master``. To działanie "rozwidla" kod i konfiguację infrastruktury.

Przechowywanie sesji w bazie danych
-----------------------------------

.. index::
    single: Session;Database

Jak można się domyślić po nazwie gałęzi, chcemy przełączyć przechowywanie sesji z systemu plików na bazę danych (stworzoną przez nas w PostgreSQL).

Kroki niezbędne do urzeczywistnienia tej koncepcji są typowe:

#. Stwórz gałąź Git;

#. W razie potrzeby zaktualizuj konfigurację Symfony;

#. Napisz i/lub zaktualizuj kod, jeśli zajdzie taka potrzeba;

#. Jeżeli potrzeba, zaktualizuj konfigurację PHP (dodaj rozszerzenie PostgreSQL w PHP);

#. Zaktualizuj infrastrukturę w Dockerze i SymfonyCloud jeżeli jest potrzeba (dodaj usługę PostgreSQL);

#. Przetestuj lokalnie;

#. Przetestuj zdalnie;

#. Połącz (ang. merge) bieżącą gałąź z gałęzią główną (ang. master);

#. Wdróż (ang. deploy) na produkcję;

#. Usuń gałąź.

Żeby przechowywać sesje w bazie danych, zmień  konfigurację ``session.handler_id`` tak, żeby wskazywała na DSN bazy:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

Żeby przechowywać sesje w bazie danych, musimy utworzyć tabele ``sessions``. Zrób to wykorzystując migracje Doctrine:

.. code-block:: bash

    $ symfony console make:migration

Zmodyfikuj plik tak, aby w metodzie ``up()`` dodać tworzenie tabeli:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,14 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs

    +        $this->addSql('
    +            CREATE TABLE sessions (
    +                sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    +                sess_data BYTEA NOT NULL,
    +                sess_lifetime INTEGER NOT NULL,
    +                sess_time INTEGER NOT NULL
    +            )
    +        ');
         }

         public function down(Schema $schema): void

Wykonaj migrację bazy danych:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Przetestuj lokalnie, poprzez przeglądanie strony internetowej. Ponieważ nie ma żadnych zmian wizualnych i nie korzystamy jeszcze z sesji, wszystko powinno działać tak jak wcześniej.

.. note::

    Nie potrzebujemy tutaj kroków od 3 do 5, ponieważ ponownie używamy bazy danych jako magazynu sesji, ale rozdział o korzystaniu z Redisa pokazuje, jak proste jest dodawanie, testowanie i wdrażanie nowej usługi zarówno w Dockerze, jak i SymfonyCloud.

Ponieważ nowa tabela nie jest „zarządzana” przez Doctrine, musimy skonfigurować Doctrine tak, aby nie usuwał jej podczas następnej migracji bazy danych:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/doctrine.yaml
    +++ b/config/packages/doctrine.yaml
    @@ -5,6 +5,8 @@ doctrine:
             # IMPORTANT: You MUST configure your server version,
             # either here or in the DATABASE_URL env var (see .env file)
             #server_version: '13'
    +
    +        schema_filter: ~^(?!session)~
         orm:
             auto_generate_proxy_classes: true
             naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware

Zatwierdź (ang. commit) swoje zmiany do nowej gałęzi (ang. branch):

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Wdrażanie gałęzi
-------------------

.. index::
    single: SymfonyCloud;Environment

Przed wdrożeniem (ang. deploy) na produkcję powinniśmy przetestować gałąź na tej samej infrastrukturze, co infrastruktura produkcyjna. Powinniśmy również sprawdzić, czy wszystko działa dobrze dla środowiska ``prod`` Symfony (lokalna strona internetowa korzystała ze środowiska ``dev`` Symfony).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

Teraz stwórzmy *środowisko SymfonyCloud* oparte na *gałęzi Git* :

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-db --no-interaction

.. code-block:: bash

    $ symfony env:create

Polecenie to tworzy nowe środowisko w następujący sposób:

* Gałąź dziedziczy kod i infrastrukturę po obecnej gałęzi Git (``sessions-in-db``);

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

Zauważ, że wszystkie polecenia SymfonyCloud działają na bieżącej gałęzi Git. Spowoduje to otwarcie adresu URL dla gałęzi ``sessions-in-db`` - adres URL będzie wyglądał tak: ``https://sessions-in-db-xxx.eu.s5y.io/``.

Przetestuj stronę internetową na tym nowym środowisku, powinny być widoczne wszystkie dane, które zostały stworzone w środowisku głównym (ang. master).

Jeśli dodasz więcej konferencji w środowisku produkcyjnym ( gałąź ``master``), nie pojawią się one w środowisku ``sessions-in-db`` i vice versa. Środowiska są niezależne i odizolowane.

Jeśli zmieni się kod w gałęzi głównej Git, zawsze można zrestartować gałąź i wdrożyć zaktualizowaną wersję, rozwiązując konflikty zarówno dla kodu, jak i infrastruktury.

.. index::
    single: Symfony CLI;env:sync

Można nawet zsynchronizować dane z gałęzi głównej (ang. master) z powrotem do środowiska ``sessions-in-db``:

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
    $ git merge sessions-in-db

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

    $ git branch -d sessions-in-db
    $ symfony env:delete --env=sessions-in-db --no-interaction

.. sidebar:: Idąc dalej

    * `Gałęzie w Gicie (ang. branching); <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_

.. _`interesariusze`: https://en.wikipedia.org/wiki/Project_stakeholder
