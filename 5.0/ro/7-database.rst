Configurarea unei baze de date
==============================

.. index::
    single: Database

„Cartea de oaspeți“ a conferinței vizează colectarea recenziilor în timpul conferințelor. Trebuie să stocăm comentariile participanților la conferință într-un mediu de stocare permanent.

Un comentariu este cel mai bine descris de o structură de date fixă: un autor, cu o adresă de email, textul recenziei și, opțional, o fotografie. Aceste date pot fi stocate foarte bine într-o bază de date relațională.

PostgreSQL este baza de date pe care o vom utiliza.

Adăugarea PostgreSQL la Docker Compose
---------------------------------------

.. index::
    single: Docker;PostgreSQL

Pe mașina noastră locală, am decis să folosim Docker pentru a gestiona serviciile. Creează un fișier ``docker-compose.yaml`` și adaugă PostgreSQL ca serviciu:

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

Aceasta va instala un server PostgreSQL la versiunea 11 și va configura unele variabile de mediu care controlează numele bazei de date și datele de autentificare. Valorile nu prea contează.

De asemenea, expunem portul PostgreSQL (``5432``) al containerului la gazda locală. Asta ne va ajuta să accesăm baza de date de pe serverul nostru.

.. note::

    Extensia ``pdo_pgsql`` ar fi trebuit să fie instalată atunci când PHP a fost configurat într-o etapă anterioară.

Pornirea Docker Compose
-----------------------

Pornește Docker Compose în fundal (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Așteaptă puțin pentru a permite pornirea bazei de date și verifică dacă totul funcționează bine:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Dacă nu există containere care rulează sau dacă în coloana ``State`` nu scrie ``Up``, atunci verifică în jurnalele Docker Compose pentru mai multe detalii:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Accesarea bazei de date locale
------------------------------

Utilizarea comenzii ``psql`` se poate dovedi utilă din când în când. Dar trebuie să îți amintești datele de autentificare și numele bazei de date. Mai puțin evident, trebuie să cunoști și portul local pe care rulează baza de date. Docker alege un port aleatoriu, astfel încât să poți lucra la mai multe proiecte folosind PotgreSQL în același timp (portul local face parte din rezultatul lui ``docker-compose ps``).

Dacă rulezi ``psql`` prin Symfony CLI, nu trebuie să memorezi nimic.

Symfony CLI detectează automat serviciile Docker care rulează pentru proiect și expune variabilele de mediu de care ``psql`` are nevoie pentru a se conecta la baza de date.

.. index::
    single: Symfony CLI;run psql

Datorită acestor convenții, accesarea bazei de date prin ``symfony run`` este mult mai ușoară:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Adăugarea PostgreSQL la SymfonyCloud
-------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Pentru infrastructura de producție de pe SymfonyCloud, adăugarea unui serviciu precum PostgreSQL trebuie făcută în fișierul actualmente gol, ``.symfony/services.yaml``:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Serviciul ``db`` este o bază de date PostgreSQL la versiunea 11 (pentru Docker) pe care dorim să o furnizăm pe un container mic cu 1 GB de spațiu disc.

De asemenea, trebuie să „conectăm” baza de date la containerul aplicației, care este descris în ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Serviciul ``db`` de tipul ``postgresql`` este declarat ``database`` în containerul aplicației.

Ultimul pas este să activezi extensia ``pdo_pgsql`` în PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Iată diferența completă pentru modificările ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

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

Salvează aceste modificări și apoi relansează proiectul în SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Accesarea bazei de date din SymfonyCloud
----------------------------------------

PostgreSQL rulează acum atât local prin Docker, cât și în producție, pe SymfonyCloud.

Așa cum tocmai am văzut, rulând ``symfony run psql``, comanda conectează proiectul automat la baza de date găzduită pe Docker datorită variabilelor de mediu expuse de ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Dacă dorești să te conectezi la PostgreSQL găzduit pe containerele de producție, poți deschide un tunel SSH între mașina locală și infrastructura SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

În mod implicit, serviciile SymfonyCloud nu sunt expuse ca variabile de mediu pe mașina locală. Trebuie să faci acest lucru în mod explicit folosind opțiunea ``--expose-env-vars``. De ce? Conectarea la baza de date de producție este o operațiune periculoasă. Poți modifica *date* reale. Folosind opțiunea confirmi că *asta* e ceea ce vrei să faci cu adevărat.

Acum, conectează-te la baza de date PostgreSQL de la distanță prin ``symfony run psql`` ca mai înainte:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Când ai terminat treaba, nu uita să închizi tunelul:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Pentru a rula câteva interogări SQL pe baza de date de producție, în loc să utilizezi un terminal direct, poți utiliza, de asemenea, comanda ``symfony sql``.

Expunerea variabilelor de mediu
-------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose și SymfonyCloud funcționează perfect cu Symfony datorită variabilelor de mediu.

Verifică toate variabilele de mediu expuse de ``symfony`` executând ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Variabilele de mediu ``PG*`` sunt citite de utilitarul ``psql``. Ce se întâmpla cu celelalte?

Când un tunel este deschis către SymfonyCloud cu opțiunea ``--expose-env-vars``, comanda ``var:export`` returnează variabilele de mediu din mediul de lucru de la distanță - de pe producție:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: Mergând mai departe

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Documentația PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `comenzi docker-compose <https://docs.docker.com/compose/reference/>`_.
