Ramificarea Codului
===================

Sunt multe modalități de a organiza fluxul modificării de cod într-un proiect. Dar lucrul direct asupra ramurei principale Git și lansarea directă în producție fără testare nu este probabil cea mai bună.

Testarea nu înseamnă doar teste unitare sau funcționale, ci și verificarea comportamentului aplicației cu date de producție. Dacă tu sau `stakeholderii`_ pot testa aplicația exact așa cum va fi pentru utilizatorii finali, acest lucru devine un avantaj uriaș și îți permite să lansezi aplicația fără temeri. Este extraordinar de util atunci când oamenii pot valida funcții noi fără să aibă nevoie de aptitudini tehnice.

O să continuăm să folosim ramura principală în Git (master), de dragul simplității și pentru a evita repetiția inutilă, dar hai să vedem cum ar fi altfel.

Adoptarea unui flux de lucru în Git
------------------------------------

Unul din fluxurile de lucru posibile este crearea unei ramuri (branch) pentru fiecare caracteristică nouă sau pentru fiecare eroare. Așa este mai simplu și eficient.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Crearea ramurilor
-----------------

.. index::
    single: Git;branch
    single: Git;checkout

Fluxul de lucru începe prin crearea unei ramuri în Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Această comandă creează o ramură ``sessions-in-redis`` din ramura ``master``. Aceasta este o „bifurcație” a codului și a configurației infrastructurii (fork).

Stocarea sesiunilor în Redis
-----------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

După cum probabil ai ghicit de la numele ramurei, dorim să schimbăm stocarea sesiunii de la sistemul de fișiere la un sistem de stocare Redis.

Pașii necesari sunt simpli:

#. Creează o ramură Git;

#. Actualizează configurația Symfony dacă este necesar;

#. Scrie și/sau actualizează codul dacă este necesar;

#. Actualizează configurația PHP (adaugă extensia PHP Redis);

#. Actualizează infrastructura de pe Docker și SymfonyCloud (adaugă serviciul Redis);

#. Testează local;

#. Testează pe serverul de la distanță;

#. Îmbină versiunea ta cu cea din ramura principală;

#. Lansează în producție;

#. Șterge ramura ta.

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

Te las să testezi local navigând pe site. Deoarece nu există modificări vizuale și pentru că nu folosim încă sesiuni, totul ar trebui să funcționeze ca înainte.

Lansarea codului dintr-o ramură
--------------------------------

.. index::
    single: SymfonyCloud;Environment

Înainte de a lansa pe producție, ar trebui să testăm ramura pe o infrastructură similară celei de producție. De asemenea, ar trebui să validăm că totul funcționează bine pentru mediul Symfony ``prod`` (site-ul local folosește mediul Symfony ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Acum, hai să creăm un mediu *SymfonyCloud* bazat pe *ramura Git*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Această comandă creează un mediu nou după cum urmează:

* Ramura moștenește codul și infrastructura de la ramura Git curentă (``sessions-in-redis``);

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

Reține că toate comenzile SymfonyCloud funcționează pe ramura Git curentă. Aceasta va deschide URL-ul implementat pentru ramura ``sessions-in-redis`` - adresa URL va arăta ca ``https://sessions-in-redis-xxx.eu.s5y.io/``).

Testează site-ul web în acest nou mediu. Ar trebui să vezi toate datele pe care le-ai creat în mediul principal.

Dacă adaugi mai multe conferințe pe mediul ``master``, acestea nu vor apărea în mediul ``sessions-in-redis`` și vice-versa. Mediile sunt independente și izolate.

Dacă codul evoluează pe master, poți întotdeauna să refaci ramura Git și să implementezi versiunea actualizată, rezolvând conflictele atât pentru cod, cât și pentru infrastructură.

.. index::
    single: Symfony CLI;env:sync

Poți chiar să sincronizezi datele din master în mediul ``sessions-in-redis``:

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
    $ git merge sessions-in-redis

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

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Mergând mai departe

    * `Ramificări în Git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`stakeholderii`: https://en.wikipedia.org/wiki/Project_stakeholder
