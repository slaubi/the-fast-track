Een database opzetten
=====================

.. index::
    single: Database

De website van het conferentiegastenboek verzamelt feedback tijdens conferenties. We moeten de reacties van deelnemers aan de conferentie ergens permanent opslaan.

Een reactie kunnen we beschrijven met een vaste datastructuur: een auteur, zijn e-mail, de tekst van de feedback en een optionele foto. Dit soort gegevens kunnen het beste opgeslagen worden in een traditioneel relationeel databasesysteem.

PostgreSQL is het database systeem dat we zullen gebruiken.

PostgreSQL aan Docker Compose toevoegen
---------------------------------------

.. index::
    single: Docker;PostgreSQL

Op onze lokale machine gebruiken we Docker om onze services te beheren. Maak een ``docker-compose.yaml`` bestand aan en voeg PostgreSQL toe als een service:

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

Dit zal een PostgreSQL server versie 11 installeren en een aantal omgevingsvariabelen configureren om de databasenaam en inloggegevens in te stellen. De waarden hiervan doen er niet direct toe.

We zetten ook de PostgreSQL-poort ( ``5432`` ) van de container open voor de lokale host. Dat zal ons toegang geven tot de database vanaf onze machine.

.. note::

    De ``pdo_pgsql`` extensie zou moeten zijn geïnstalleerd toen PHP opgezet werd in een vorige stap.

Docker Compose starten
----------------------

Start Docker Compose in de achtergrond ( ``-d`` ):

.. code-block:: bash

    $ docker-compose up -d

Wacht even tot de database volledig opgestart is en controleer dan of alles goed draait:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Als er geen containers draaien of als de ``State`` kolom niet ``Up`` bevat, bekijk dan de Docker Compose logs:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

De lokale database benaderen
----------------------------

Het ``psql`` command-line hulpprogramma kan soms van pas komen. Je moet dan wel de inloggegevens en de naam van de database onthouden. Maar iets minder handig is dat je ook moet weten op welke lokale poort de database op de host draait. Docker kiest hiervoor namelijk een willekeurige poort, zodat je aan meer dan één project tegelijk met PostgreSQL kunt werken (de lokale poort kan je vinden via het ``docker-compose ps`` commando).

Als je ``psql`` via de Symfony CLI draait, hoef je niets te onthouden.

De Symfony CLI detecteert automatisch de Docker-services die voor het project draaien en stelt de omgevingsvariabelen in die ``psql`` nodig heeft om een verbinding te maken met de database.

.. index::
    single: Symfony CLI;run psql

Dankzij deze conventies is het benaderen van de database via ``symfony run`` veel eenvoudiger:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

PostgreSQL toevoegen aan SymfonyCloud
-------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Een service zoals PostgreSQL toevoegen aan de productie-infrastructuur op SymfonyCloud doe je door deze toe te voegen in het ``.symfony/services.yaml`` bestand. Dit bestand is momenteel nog leeg:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

De ``db`` service is een PostgreSQL versie 11 database (zoals we in Docker gebruiken) die we willen draaien in een kleine container met 1GB schijfruimte.

We moeten de DB ook "koppelen" aan de applicatie-container die beschreven staat in ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

In de applicatie-container wordt er aan de ``db`` service van het type ``postgresql`` gerefereerd met de naam ``database``.

Als laatste stap voegen we de ``pdo_pgsql`` extensie nog toe aan de PHP-runtime:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Hier is de volledige diff van de wijzigingen aan ``.symfony.cloud.yaml``:

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

Commit deze wijzigingen en deploy ze opnieuw naar SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Toegang tot de SymfonyCloud database
------------------------------------

PostgreSQL draait nu zowel lokaal via Docker als in productie op SymfonyCloud.

Zoals we zojuist hebben gezien kun je met het ``symfony run psql`` commando, automatisch een verbinding maken naar de database die door Docker wordt gehost, dit alles dankzij de omgevingsvariabelen die door ``symfony run`` werden ingesteld.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Als je verbinding wil maken met PostgreSQL in de containers van de productieomgeving, kun je een SSH tunnel leggen tussen de lokale machine en de SymfonyCloud infrastructuur:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

SymfonyCloud services worden standaard niet via omgevingsvariabelen op de lokale machine ingesteld. Je moet dit expliciet doen door de ``--expose-env-vars`` parameter te gebruiken. Waarom? Verbinding maken met een productiedatabase is gevaarlijk. Je kan *echte* data om zeep helpen. Door het gebruik van de parameter, geef je aan dat dit *echt* de bedoeling is.

Verbind nu met de externe PostgreSQL database via ``symfony run psql``:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Vergeet niet de tunnel te sluiten wanneer je klaar bent:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Om wat SQL queries uit te voeren op de productiedatabase, kan je ook het ``symfony sql`` commando gebruiken, in plaats van een shell te starten.

Werken met omgevingsvariabelen
------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose en SymfonyCloud werken naadloos samen met Symfony dankzij het gebruik van omgevingsvariabelen.

Controleer alle omgevingsvariabelen die door ``symfony`` gebruikt worden via ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

De ``PG*`` omgevingsvariabelen worden gebruikt door het ``psql`` hulpprogramma. En de rest?

Wanneer er een tunnel ligt naar SymfonyCloud via de ``--expose-env-vars`` parameter, geeft het ``var:export`` commando de externe omgevingsvariabelen weer:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: Verder gaan

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `PostgreSQL documentatie <https://www.postgresql.org/docs/>`_;

    * `docker-compose commando's <https://docs.docker.com/compose/reference/>`_.
