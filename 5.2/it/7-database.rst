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

Sulla nostra macchina locale, abbiamo deciso di utilizzare Docker per gestire i servizi. Creare un file ``docker-compose.yaml`` e aggiungere PostgreSQL come servizio:

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

Questo installerà un server PostgreSQL e configurerà alcune variabili d'ambiente, che controllano nome e credenziali del database. I valori non hanno molta importanza.

Esponiamo anche la porta PostgreSQL (``5432``) del container all'host locale. Questo ci aiuterà ad accedere al database dalla nostra macchina.

.. note::

    L'estensione ``pdo_pgsql`` dovrebbe essere stata installata quando PHP è stato impostato, in uno dei passi precedenti.

Avvio di Docker Compose
-----------------------

Avviare Docker Compose in background (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Attendere un po' per far partire il database e verificare che tutto funzioni correttamente:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Se non ci sono container in esecuzione o se la  colonna ``State`` non mostra la scritta ``Up``, controllare i log di Docker Compose:

.. code-block:: bash
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

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    Se non si dispone del binario ``psql`` sull'host locale, è possibile eseguirlo anche tramite ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Esportare e importare dati dal database
---------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Usare ``pg_dump`` per esportare i dati dal database:

.. code-block:: bash
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

E importare i dati:

.. code-block:: bash
    :class: ignore

    $ symfony run psql < dump.sql

.. warning::

    Non richiamare mai ``docker-compose down``, se non si vogliono perdere dati. Oppure farne prima un backup.

Aggiungere PostgreSQL a SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Per l'infrastruttura di produzione su SymfonyCloud, l'aggiunta di un servizio come PostgreSQL dovrebbe essere fatta nel file, attualmente vuoto, ``.symfony/services.yaml``:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Il servizio ``db`` è un database PostgreSQL (stessa versione di Docker), che vogliamo mettere a disposizione su un piccolo container con 1GB di disco.

Abbiamo anche bisogno di "collegare" il DB al container dell'applicazione, che è descritto in ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Il servizio ``db`` di tipo ``postgresql`` si riferisce al container ``database`` dell'applicazione.

L'ultimo passo è quello di aggiungere l'estensione ``pdo_pgsql`` a PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Ecco tutte le modifiche di ``.symfony.cloud.yaml``:

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

Fare commit di queste modifiche e poi rifare il deploy su SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Accesso al database di SymfonyCloud
-----------------------------------

PostgreSQL è ora in esecuzione sia localmente tramite Docker sia in produzione su SymfonyCloud.

Come abbiamo appena visto, eseguendo ``symfony run psql`` si connette automaticamente al database ospitato da Docker, grazie alle variabili d'ambiente esposte da ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Se ci si vuole connettere a PostgreSQL ospitato sui container di produzione, si può aprire un tunnel SSH tra la macchina locale e l'infrastruttura SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

Per impostazione predefinita, i servizi di SymfonyCloud non sono esposti come variabili d'ambiente sulla macchina locale. È necessario farlo esplicitamente utilizzando il parametro ``--expose-env-vars``. Perché? Il collegamento al database di produzione è un'operazione pericolosa. Si possono mettere a repentaglio i dati *reali*. Richiedere questa opzione è come confermare che *effettivamente* l'azione è quella desiderata.

Ora, connettersi al database PostgreSQL remoto tramite ``symfony run psql``, come in precedenza:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Al termine, non dimenticare di chiudere il tunnel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Per eseguire query SQL sul database di produzione invece di usare una shell, si può anche usare il comando ``symfony sql``.

Esporre le variabili d'ambiente
-------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose e SymfonyCloud funzionano perfettamente con Symfony, grazie alle variabili d'ambiente.

Controllare tutte le variabili d'ambiente ``symfony`` esposte, eseguendo ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Le variabili d' ambiente ``PG*`` vengono lette dal comando ``psql``. E le altre?

Quando si apre un tunnel verso SymfonyCloud usando il parametro ``--expose-env-vars``, il comando ``var:export`` restituisce le variabili d'ambiente remote:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

Descrivere l'infrastruttura
---------------------------

Potrebbe non essere evidente all'inizio, ma avere l'infrastruttura memorizzata in dei file, insieme al codice, aiuta molto. Docker e SymfonyCloud usano dei file per descrivere l'infrastruttura del progetto. Quando una nuova funzionalità necessita di un servizio aggiuntivo, le modifiche al codice e quelle all'infrastruttura andranno così di pari passo.

.. sidebar:: Andare oltre

    * `Servizi di SymfonyCloud <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Documentazione PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `Comandi docker-compose <https://docs.docker.com/compose/reference/>`_.
