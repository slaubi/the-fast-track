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

Op onze lokale machine gebruiken we Docker om onze services te beheren. Het gegenereerde ``compose.yaml`` bestand bevat reeds PostgreSQL als service:

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

Dit zal een PostgreSQL server installeren en een aantal omgevingsvariabelen configureren om de databasenaam en inloggegevens in te stellen. De waarden hiervan doen er niet direct toe.

We zetten ook de PostgreSQL-poort (``5432``) van de container open voor de lokale host. Dat zal ons toegang geven tot de database vanaf onze machine:

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

    De ``pdo_pgsql`` extensie zou moeten zijn geïnstalleerd toen PHP opgezet werd in een vorige stap.

Docker Compose starten
----------------------

Start Docker Compose in de achtergrond ( ``-d`` ):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

Wacht even tot de database volledig opgestart is en controleer dan of alles goed draait:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Als er geen containers draaien of als de ``State`` kolom niet ``Up`` bevat, bekijk dan de Docker Compose logs:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

De lokale database benaderen
----------------------------

Het ``psql`` command-line hulpprogramma kan soms van pas komen. Je moet dan wel de inloggegevens en de naam van de database onthouden. Maar iets minder handig is dat je ook moet weten op welke lokale poort de database op de host draait. Docker kiest hiervoor namelijk een willekeurige poort, zodat je aan meer dan één project tegelijk met PostgreSQL kunt werken (de lokale poort kan je vinden via het ``docker compose ps`` commando).

Als je ``psql`` via de Symfony CLI draait, hoef je niets te onthouden.

De Symfony CLI detecteert automatisch de Docker-services die voor het project draaien en stelt de omgevingsvariabelen in die ``psql`` nodig heeft om een verbinding te maken met de database.

.. index::
    single: Symfony CLI;run psql

Dankzij deze conventies is het benaderen van de database via ``symfony run`` veel eenvoudiger:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Als de ``psql`` binary niet aanwezig is op jouw lokale host kun je deze ook uitvoeren via ``docker compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

Database data dumpen en terugzetten
-----------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Gebruik ``pg_dump`` om de database data te dumpen:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

En zet de data terug:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

PostgreSQL toevoegen aan Upsun
------------------------------------

.. index::
    single: Upsun;PostgreSQL

Voor de productie infrastructuur op Upsun, zou je een service zoals PostgreSQL moeten toevoegen in het ``.upsun/config.yaml`` bestand, wat reeds gedaan werd door het recept van de ``webapp`` package:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

De ``database`` service is een PostgreSQL database (dezelfde versie als we in Docker gebruiken). Upsun wijst de schijfruimte automatisch toe bij de eerste deployment; pas deze later aan met ``symfony cloud:resources:set`` indien nodig.

We moeten de DB ook "koppelen" aan de applicatie-container die beschreven staat in ``.upsun/config.yaml``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

In de applicatie-container wordt er aan de ``database`` service van het type ``postgresql`` gerefereerd met de naam ``database``.

Controleer of de ``pdo_pgsql`` extensie reeds geïnstalleerd is voor de PHP-runtime:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Toegang tot de Upsun database
-----------------------------------

PostgreSQL draait nu zowel lokaal via Docker als in productie op Upsun.

Zoals we zojuist hebben gezien kun je met het ``symfony run psql`` commando, automatisch een verbinding maken naar de database die door Docker wordt gehost, dit alles dankzij de omgevingsvariabelen die door ``symfony run`` werden ingesteld.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Als je verbinding wil maken met PostgreSQL in de containers van de productieomgeving, kun je een SSH tunnel leggen tussen de lokale machine en de Upsun infrastructuur:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Upsun services worden standaard niet via omgevingsvariabelen op de lokale machine ingesteld. Je moet dit expliciet doen door het ``var:expose-from-tunnel`` commando te draaien. Waarom? Verbinding maken met een productiedatabase is gevaarlijk. Je kan *echte* data om zeep helpen.

Verbind nu met de externe PostgreSQL database via ``symfony run psql``:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Vergeet niet de tunnel te sluiten wanneer je klaar bent:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Om wat SQL queries uit te voeren op de productiedatabase, kan je ook het ``symfony sql`` commando gebruiken, in plaats van een shell te starten.

Werken met omgevingsvariabelen
------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

Docker Compose en Upsun werken naadloos samen met Symfony dankzij het gebruik van omgevingsvariabelen.

Controleer alle omgevingsvariabelen die door ``symfony`` gebruikt worden via ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

De ``PG*`` omgevingsvariabelen worden gebruikt door het ``psql`` hulpprogramma. En de rest?

Wanneer er een tunnel ligt naar Upsun via ``var:expose-from-tunnel``, geeft het ``var:export`` commando de externe omgevingsvariabelen weer:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Je Infrastructuur omschrijven
-----------------------------

Je hebt het misschien nog niet gerealiseerd, maar het helpt veel om de infrastructuur naast de code in bestanden op te slaan. Docker en Upsun gebruiken configuratiebestanden om de project-infrastructuur te omschrijven. Wanneer een nieuwe functie een aanvullende service nodig heeft, veranderen de code en maken de wijzigingen in de infrastructuur deel uit van dezelfde patch.

.. sidebar:: Verder gaan

    * `Upsun diensten`_;

    * `Upsun tunnel`_;

    * `PostgreSQL documentatie`_;

    * `Docker Compose commando's`_.

.. _`Upsun diensten`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Upsun tunnel`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL documentatie`: https://www.postgresql.org/docs/
.. _`Docker Compose commando's`: https://docs.docker.com/reference/cli/docker/compose/
