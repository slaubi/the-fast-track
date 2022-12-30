Impostare il database
=====================

.. index::
    single: Database

Il sito web del guestbook della conferenza è dedicato alla raccolta di feedback durante le conferenze. Abbiamo bisogno di memorizzare in modo permanente i commenti dei partecipanti.

Un commento può essere descritto da una struttura dati fissa: un autore, la sua email, il testo del feedback e una foto opzionale. Questo tipo di dati si presta a essere memorizzato in un classico database relazionale.

PostgreSQL è il motore di database che useremo.

Aggiunta di PostgreSQL a Docker Compose
---------------------------------------

.. index::
    single: Docker;PostgreSQL

Sulla nostra macchina locale, abbiamo deciso di utilizzare Docker per gestire i servizi. Il file generato ``docker-compose.yaml`` contiene gìà PostgreSQL come servizio:

.. code-block:: yaml
    :caption: docker-compose.yml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-14}-alpine
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

Questo installerà un server PostgreSQL e configurerà alcune variabili d'ambiente, che controllano nome e credenziali del database. I valori non hanno molta importanza.

Esponiamo anche la porta di PostgreSQL (``5432``) del container all'host locale. Questo ci aiuterà ad accedere al database dalla nostra macchina:

.. code-block:: yaml
    :caption: docker-compose.override.yml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    L'estensione ``pdo_pgsql`` dovrebbe essere stata installata quando PHP è stato impostato, in uno dei passi precedenti.

Avvio di Docker Compose
-----------------------

Avviare Docker Compose in background (``-d``):

.. code-block:: terminal

    $ docker-compose up -d

Attendere un po' per far partire il database e verificare che tutto funzioni correttamente:

.. code-block:: terminal
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Se non ci sono container in esecuzione o se la  colonna ``State`` non mostra la scritta ``Up``, controllare i log di Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker-compose logs

Accesso al database locale
--------------------------

L'utilizzo del programma a riga di comando ``psql`` potrebbe rivelarsi utile di tanto in tanto. Occorre però ricordare le credenziali e il nome del database. Meno ovvio, occorre anche conoscere la porta locale con cui il database gira sull'host. Docker sceglie una porta casuale in modo da poter lavorare su più di un progetto contemporaneamente utilizzando PostgreSQL (la porta locale è parte dell'output di ``docker-compose ps``).

Se si esegue ``psql`` tramite la CLI di Symfony, non è necessario ricordare nulla.

La CLI di Symfony rileva automaticamente i servizi Docker in esecuzione per il progetto ed espone le variabili d'ambiente di cui ``psql`` ha bisogno per connettersi al database.

.. index::
    single: Symfony CLI;run psql

Grazie a queste convenzioni, l'accesso al database tramite ``symfony run`` è molto più semplice:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Se non si dispone del binario ``psql`` sull'host locale, è possibile eseguirlo anche tramite ``docker-compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker-compose exec database psql app app

Esportare e importare dati dal database
---------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Usare ``pg_dump`` per esportare i dati dal database:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

E importare i dati:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Aggiungere PostgreSQL a Platform.sh
-----------------------------------

.. index::
    single: Platform.sh;PostgreSQL

Per l'infrastruttura di produzione su Platform.sh, l'aggiunta di un servizio come PostgreSQL dovrebbe essere fatta nel file ``.platform/services.yaml``, cosa che è già stata fatta attraverso la ricetta del pacchetto ``webapp``:

.. code-block:: yaml
    :caption: .platform/services.yaml
    :class: ignore

    database:
        type: postgresql:14
        disk: 1024

Il servizio ``database`` è un database PostgreSQL (stessa versione di Docker), che vogliamo configurare con 1GB di spazio.

Abbiamo anche bisogno di "collegare" il DB al container dell'applicazione, come descritto nel file ``.platform.app.yaml``:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Il servizio ``database`` di tipo ``postgresql`` si riferisce al container ``database`` dell'applicazione.

Controllare che l'estensione `pdo_pgsql`` sia installata per PHP:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Accesso al database di Platform.sh
----------------------------------

PostgreSQL è ora in esecuzione sia localmente tramite Docker sia in produzione su Platform.sh.

Come abbiamo appena visto, eseguendo ``symfony run psql`` si connette automaticamente al database ospitato da Docker, grazie alle variabili d'ambiente esposte da ``symfony run``.

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Se ci si vuole connettere a PostgreSQL ospitato sui container di produzione, si può aprire un tunnel SSH tra la macchina locale e l'infrastruttura Platform.sh:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Per impostazione predefinita, i servizi di Platform.sh non sono esposti come variabili d'ambiente sulla macchina locale. È necessario farlo esplicitamente utilizzando il parametro ``--expose-env-vars``. Perché? Il collegamento al database di produzione è un'operazione pericolosa. Si possono mettere a repentaglio i dati *reali*.

Ora, connettersi al database PostgreSQL remoto tramite ``symfony run psql``, come in precedenza:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Al termine, non dimenticare di chiudere il tunnel:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Per eseguire query SQL sul database di produzione invece di usare una shell, si può anche usare il comando ``symfony sql``.

Esporre le variabili d'ambiente
-------------------------------

.. index::
    single: Platform.sh;Environment Variables
    single: Symfony CLI;var:export

Docker Compose e Platform.sh funzionano perfettamente con Symfony, grazie alle variabili d'ambiente.

Controllare tutte le variabili d'ambiente ``symfony`` esposte, eseguendo ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Le variabili d' ambiente ``PG*`` vengono lette dal comando ``psql``. E le altre?

Quando si apre un tunnel verso Platform.sh usando ``var:expose-from-tunnel``, il comando ``var:export`` restituisce le variabili d'ambiente remote:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Descrivere l'infrastruttura
---------------------------

Potrebbe non essere evidente all'inizio, ma avere l'infrastruttura memorizzata in dei file, insieme al codice, aiuta molto. Docker e Platform.sh usano dei file per descrivere l'infrastruttura del progetto. Quando una nuova funzionalità necessita di un servizio aggiuntivo, le modifiche al codice e quelle all'infrastruttura andranno così di pari passo.

.. sidebar:: Andare oltre

    * `Servizi di Platform.sh`_;

    * `Platform.sh tunnel`_;

    * `Documentazione PostgreSQL`_;

    * `Comandi docker-compose`_.

.. _`Servizi di Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Platform.sh tunnel`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Documentazione PostgreSQL`: https://www.postgresql.org/docs/
.. _`Comandi docker-compose`: https://docs.docker.com/compose/reference/
