Ramificarea Codului
===================

Sunt multe modalități de a organiza fluxul modificării de cod într-un proiect. Dar lucrul direct asupra ramurei principale Git și lansarea directă în producție fără testare nu este probabil cea mai bună.

Testarea nu înseamnă doar teste unitare sau funcționale, ci și verificarea comportamentului aplicației cu date de producție. Dacă tu sau `stakeholderii`_ pot testa aplicația exact așa cum va fi pentru utilizatorii finali, acest lucru devine un avantaj uriaș și îți permite să lansezi aplicația fără temeri. Este extraordinar de util atunci când oamenii pot valida funcții noi fără să aibă nevoie de aptitudini tehnice.

O să continuăm să folosim ramura principală în Git (master), de dragul simplității și pentru a evita repetiția inutilă, dar hai să vedem cum ar fi altfel.

Adoptarea unui flux de lucru în Git
------------------------------------

Unul din fluxurile de lucru posibile este crearea unei ramuri (branch) pentru fiecare caracteristică nouă sau pentru fiecare eroare. Așa este mai simplu și eficient.

Crearea ramurilor
-----------------

.. index::
    single: Git;branch
    single: Git;checkout

Fluxul de lucru începe prin crearea unei ramuri în Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: bash

    $ git checkout -b sessions-in-db

Această comandă creează o ramură ``sessions-in-db`` din ramura ``master``. Aceasta este o „bifurcație” a codului și a configurației infrastructurii (fork).

Stocarea sesiunilor în baza de date
------------------------------------

.. index::
    single: Session;Database

După cum probabil ai ghicit de la numele ramurei, dorim să schimbăm stocarea sesiunii de la sistemul de fișiere la un sistem de stocare în baza de date (baza noastră de date PostgreSQL de aici).

Pașii necesari sunt simpli:

#. Creează o ramură Git;

#. Actualizează configurația Symfony dacă este necesar;

#. Scrie și/sau actualizează codul dacă este necesar;

#. Actualizează configurația PHP dacă este necesar (precum adăugarea extensiei PHP PostgreSQL);

#. Actualizează infrastructura de pe Docker și SymfonyCloud dacă este necesar (adaugă serviciul PostgreSQL);

#. Testează local;

#. Testează pe serverul de la distanță;

#. Îmbină versiunea ta cu cea din ramura principală;

#. Lansează în producție;

#. Șterge ramura ta.

Pentru a stoca sesiuni în baza de date, modifică configurația ``session.handler_id`` pentru a indica către DSN-ul bazei de date:

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

Pentru a stoca sesiuni în baza de date, trebuie să creăm tabelul ``sessions``. Fă acest lucru cu o migrare Doctrine:

.. code-block:: bash

    $ symfony console make:migration

Editează fișierul pentru a adăuga crearea tabelului în metoda ``up()``:

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

Migrează baza de date:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Testează local navigând pe site. Deoarece nu există modificări vizuale și pentru că nu folosim încă sesiuni, totul ar trebui să funcționeze ca înainte.

.. note::

    Nu avem nevoie de pașii de la 3 la 5 aici, deoarece reutilizăm baza de date pentru stocarea sesiunii, dar capitolul despre utilizarea Redis-ului arată cât de simplu este să adaugi, să testezi și să pui pe server un nou serviciu atât în Docker cât și în SymfonyCloud

Deoarece noul tabel nu este „gestionat” de Doctrine, trebuie să configurăm Doctrine să nu-l șteargă la următoarea migrare a bazei de date:

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

Salvează modificările într-o ramură nouă:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Lansarea codului dintr-o ramură
--------------------------------

.. index::
    single: SymfonyCloud;Environment

Înainte de a lansa pe producție, ar trebui să testăm ramura pe o infrastructură similară celei de producție. De asemenea, ar trebui să validăm că totul funcționează bine pentru mediul Symfony ``prod`` (site-ul local folosește mediul Symfony ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

Acum, hai să creăm un mediu *SymfonyCloud* bazat pe *ramura Git*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-db --no-interaction

.. code-block:: bash

    $ symfony env:create

Această comandă creează un mediu nou după cum urmează:

* Ramura moștenește codul și infrastructura de la ramura Git curentă (``sessions-in-db``);

* Datele provin din mediul principal (adică cea folosită în producție), salvând o copie completă a tuturor datelor, inclusiv fișiere (fișierele încărcate de utilizatori, de exemplu) și baze de date;

* Un nou cluster dedicat este creat pentru a lansa codul, datele și infrastructura.

Întrucât lansarea curentă urmează aceleași etape ca lansarea ulterioară pentru producție, migrațiile bazei de date vor fi, de asemenea, executate. În acest fel, vei valida că migrațiile funcționează cu datele din producție.

Mediile care nu sunt ``master`` sunt foarte asemănătoare cu ``master``, cu excepția unor mici diferențe: de exemplu, e-mailurile nu sunt trimise implicit; trimiterea lor e doar simulată.

.. index::
    single: Symfony CLI;open:remote

După terminarea implementării, deschide site-ul într-un browser pentru a testa noua ramură:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Reține că toate comenzile SymfonyCloud funcționează pe ramura Git curentă. Aceasta va deschide URL-ul implementat pentru ramura ``sessions-in-db`` - adresa URL va arăta ca ``https://sessions-in-db-xxx.eu.s5y.io/``).

Testează site-ul web în acest nou mediu. Ar trebui să vezi toate datele pe care le-ai creat în mediul principal.

Dacă adaugi mai multe conferințe pe mediul ``master``, acestea nu vor apărea în mediul ``sessions-in-db`` și vice-versa. Mediile sunt independente și izolate.

Dacă codul evoluează pe master, poți întotdeauna să refaci ramura Git și să implementezi versiunea actualizată, rezolvând conflictele atât pentru cod, cât și pentru infrastructură.

.. index::
    single: Symfony CLI;env:sync

Poți chiar să sincronizezi datele din master în mediul ``sessions-in-db``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Depanarea implementărilor de producție înainte de lansare
------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

În mod implicit, toate mediile SymfonyCloud folosesc aceleași setări ca mediul ``master``/``prod`` (denumit, de asemenea, mediul Symfony ``prod``). Acest lucru îți permite să testezi aplicația în condiții reale. Îți creează impresia că lucrezi direct pe serverele de producție, dar fără riscurile asociate. Acest lucru îmi amintește de vremurile bune când publicam proiectele prin FTP.

.. index::
    single: Symfony CLI;env:debug

În cazul unei probleme, poți să treci la mediul ``dev`` Symfony:

.. code-block:: bash

    $ symfony env:debug

După ce ai terminat, te întorci la setările de producție:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Niciodată** nu activa mediul ``dev`` sau Depanatorul Symfony în ramura ``master``; ar face ca aplicația ta să devină lentă și deschide o mulțime de vulnerabilități grave de securitate.

Testarea implementărilor de producție înainte de publicare
-------------------------------------------------------------

Accesul la versiunea viitoare a site-ului web cu date de producție deschide o mulțime de oportunități: de la testarea regresiei vizuale la testarea performanței. `Blackfire <https://blackfire.io>`_ este instrumentul perfect pentru job.

Consultă pasul despre „performanță” pentru a afla mai multe despre modul în care poți utiliza Blackfire pentru a testa codul tău înainte de publicare.

Fuziunea cu producția
----------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Când ești mulțumit de modificările din ramura ta, îmbină modificările din cod și infrastructură înapoi în ramura principală Git printr-o fuziune:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-db

Și publică:

.. code-block:: bash

    $ symfony deploy

Când se lansează, doar modificările codului și infrastructurii sunt împinse către SymfonyCloud; datele nu sunt afectate în niciun fel.

Curățenie
-----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

La final, facem curățenie ștergând ramura din Git și mediul din SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-db
    $ symfony env:delete --env=sessions-in-db --no-interaction

.. sidebar:: Mergând mai departe

    * `Ramificări în Git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

.. _`stakeholderii`: https://en.wikipedia.org/wiki/Project_stakeholder
