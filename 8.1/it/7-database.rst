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

Sulla nostra macchina locale, abbiamo deciso di utilizzare Docker per gestire i servizi. Il file generato ``compose.yaml`` contiene gìà PostgreSQL come servizio:

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

Questo installerà un server PostgreSQL e configurerà alcune variabili d'ambiente, che controllano nome e credenziali del database. I valori non hanno molta importanza.

Esponiamo anche la porta di PostgreSQL (``5432``) del container all'host locale. Questo ci aiuterà ad accedere al database dalla nostra macchina:

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

    L'estensione ``pdo_pgsql`` dovrebbe essere stata installata quando PHP è stato impostato, in uno dei passi precedenti.

Avvio di Docker Compose
-----------------------

Avviare Docker Compose in background (``-d``):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

Attendere un po' per far partire il database e verificare che tutto funzioni correttamente:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Se non ci sono container in esecuzione o se la  colonna ``State`` non mostra la scritta ``Up``, controllare i log di Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

Accesso al database locale
--------------------------

Using the ``psql`` command-line utility might prove useful from time to time. But you need to remember the credentials and the database name. Less obvious, you also need to know the local port the database runs on the host. Docker chooses a random port so that you can work on more than one project using PostgreSQL at the same time (the local port is part of the output of ``docker compose ps``).

Se si esegue ``psql`` tramite la CLI di Symfony, non è necessario ricordare nulla.

La CLI di Symfony rileva automaticamente i servizi Docker in esecuzione per il progetto ed espone le variabili d'ambiente di cui ``psql`` ha bisogno per connettersi al database.

.. index::
    single: Symfony CLI;run psql

Grazie a queste convenzioni, l'accesso al database tramite ``symfony run`` è molto più semplice:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Se non si dispone del binario ``psql`` sull'host locale, è possibile eseguirlo anche tramite ``docker compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

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

Aggiungere PostgreSQL a Upsun
-----------------------------------

.. index::
    single: Upsun;PostgreSQL

Per l'infrastruttura di produzione su Upsun, l'aggiunta di un servizio come PostgreSQL dovrebbe essere fatta nel file ``.upsun/config.yaml``, cosa che è già stata fatta attraverso la ricetta del pacchetto ``webapp``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

Il servizio ``database`` è un database PostgreSQL (stessa versione di Docker). Upsun assegna automaticamente il suo spazio su disco al primo deploy; all'occorrenza è possibile modificarlo in seguito con ``symfony cloud:resources:set``.

Abbiamo anche bisogno di "collegare" il DB al container dell'applicazione, come descritto nel file ``.upsun/config.yaml``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Il servizio ``database`` di tipo ``postgresql`` si riferisce al container ``database`` dell'applicazione.

Controllare che l'estensione `pdo_pgsql`` sia installata per PHP:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Accesso al database di Upsun
----------------------------------

PostgreSQL è ora in esecuzione sia localmente tramite Docker sia in produzione su Upsun.

Come abbiamo appena visto, eseguendo ``symfony run psql`` si connette automaticamente al database ospitato da Docker, grazie alle variabili d'ambiente esposte da ``symfony run``.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Se ci si vuole connettere a PostgreSQL ospitato sui container di produzione, si può aprire un tunnel SSH tra la macchina locale e l'infrastruttura Upsun:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Per impostazione predefinita, i servizi di Upsun non sono esposti come variabili d'ambiente sulla macchina locale. È necessario farlo esplicitamente utilizzando il comando ``var:expose-from-tunnel``. Perché? Il collegamento al database di produzione è un'operazione pericolosa. Si possono mettere a repentaglio i dati *reali*.

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
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

Docker Compose e Upsun funzionano perfettamente con Symfony, grazie alle variabili d'ambiente.

Controllare tutte le variabili d'ambiente ``symfony`` esposte, eseguendo ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Le variabili d' ambiente ``PG*`` vengono lette dal comando ``psql``. E le altre?

Quando si apre un tunnel verso Upsun usando ``var:expose-from-tunnel``, il comando ``var:export`` restituisce le variabili d'ambiente remote:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Descrivere l'infrastruttura
---------------------------

Potrebbe non essere evidente all'inizio, ma avere l'infrastruttura memorizzata in dei file, insieme al codice, aiuta molto. Docker e Upsun usano dei file per descrivere l'infrastruttura del progetto. Quando una nuova funzionalità necessita di un servizio aggiuntivo, le modifiche al codice e quelle all'infrastruttura andranno così di pari passo.

.. sidebar:: Andare oltre

    * `Servizi di Upsun`_;

    * `Tunnel Upsun`_;

    * `Documentazione PostgreSQL`_;

    * i `comandi di Docker Compose`_.

.. _`Servizi di Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Tunnel Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Documentazione PostgreSQL`: https://www.postgresql.org/docs/
.. _`comandi di Docker Compose`: https://docs.docker.com/reference/cli/docker/compose/
