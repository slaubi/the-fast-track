Creare branch del codice
========================

Esistono molti modi per organizzare il flusso di lavoro per modificare il codice di un progetto, ma lavorare sul branch master ed effettuare un deploy direttamente in produzione del codice senza una suite di test, probabilmente, non è la scelta migliore.

Testare l'applicazione non riguarda solo i test unitari o funzionali, ma anche il controllo del comportamento dell'applicazione con i dati di produzione. Se voi o i vostri `stakeholder`_ potete utilizzare l'applicazione esattamente come sarà distribuita agli utenti finali, questo sarà un enorme vantaggio e vi permetterà di effettuare deploy con più sicurezza. Mettere in condizione delle persone non tecniche di validare le nuove funzionalità è un concetto importantissimo.

Per semplicità e per evitare di ripeterci continueremo a lavorare nel branch master, ma più avanti vedremo come migliorare questo aspetto.

Adottare un flusso di lavoro con git
------------------------------------

Un possibile flusso di lavoro potrebbe essere quello di creare un branch per ogni nuova funzionalità o la risoluzione di un bug. Quest'approccio è semplice ed efficiente.

Creazione di branch
-------------------

.. index::
    single: Git;branch
    single: Git;checkout

Il flusso di lavoro inizia con la creazione di un nuovo branch git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: bash

    $ git checkout -b sessions-in-db

Questo comando crea un nuovo branch dal nome ``sessions-in-db`` partendo dal branch ``master``. Sarà una "biforcazione" del codice e della configurazione dell'infrastruttura.

Salvare le sessioni nel database
--------------------------------

.. index::
    single: Session;Database

Come probabilmente avrete intuito dal nome del branch, vogliamo cambiare il salvataggio delle sessioni, togliendole dal filesystem per memorizzarle nel database (il nostro caso il database PostgreSQL).

I passi necessari per fare questo sono semplicemente:

#. Creare un branch git;

#. Aggiornare la configurazione di Symfony, se necessario;

#. Aggiungere e/o modificare qualche linea di codice se necessario;

#. Aggiornare la configurazione di PHP se necessario (aggiungendo ad esempio l'estensione PostgreSQL per PHP);

#. Aggiornare l'infrastruttura su Docker e SymfonyCloud se necessario (aggiungendo il servizio PostgreSQL);

#. Testare nel nostro ambiente in locale;

#. Effettuare dei test anche da remoto;

#. Effettuare un merge del branch su master;

#. Deploy in produzione;

#. Cancellare il branch.

Per salvare le sessioni nel database, cambiare il valore di  ``session.handler_id`` per puntare al DSN del database:

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

Per salvare le sessioni sul database, dobbiamo creare la tabella ``sessions``. Possiamo farlo con una migrazione di Doctrine:

.. code-block:: bash

    $ symfony console make:migration

Modifichiamo il file aggiungendo la creazione della tabella nel metodo ``up()``:

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

Eseguiamo le migrazioni del database:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Controlliamo il sito in locale. Poiché non ci sono cambiamenti visivi e poiché non stiamo ancora usando le sessioni, dovrebbe funzionare tutto come prima.

.. note::

    Non abbiamo bisogno dei passi da 3 a 5, poiché stiamo riutilizzando il database per salvare le sessioni, ma il capitolo su Redis mostra quanto sia facile aggiungere, testare e mettere in produzione un nuovo servizio con Docker e SymfonyCloud.

Non esendo questa nuova tabella gestita da Doctrine, dobbiamo configurare Doctrine per non eliminarla nella prossima migrazione:

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

Eseguiamo il commit delle modifiche sul nuovo branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Deploy di un branch
-------------------

.. index::
    single: SymfonyCloud;Environment

Prima di passare al deploy in produzione, dovremmo testare il branch su un'infrastruttura identica a quella di produzione. Dovremmo anche controllare che tutto funzioni bene sull'ambiente Symfony ``prod`` (il sito locale usava l'ambiente Symfony ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

Ora, creiamo un *ambiente SymfonyCloud* a partire dal *branch*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-db --no-interaction

.. code-block:: bash

    $ symfony env:create

Con il seguente comando creeremo un nuovo ambiente:

* Il branch eredita il codice e l'infrastruttura dall'attuale branch (``sessions-in-db``);

* I dati provengono dall'ambiente master (ovvero dalla produzione) prendendo un'istantanea coerente di tutti i dati come i file (quelli caricati dagli utenti, per esempio) e i database;

* Viene creato un nuovo cluster dedicato per effettuare il deploy del codice, per i dati e per l'infrastruttura.

Poiché il deploy segue gli stessi passi del deploy in produzione, verranno eseguite anche le migrazioni del database. Questo è un ottimo modo per convalidare che le migrazioni funzionino con i dati di produzione.

Gli ambienti differenti da ``master`` sono molto simili a quello ``master``, a eccezione di alcune piccole differenze: ad esempio, per impostazione predefinita, le email non vengono inviate.

.. index::
    single: Symfony CLI;open:remote

Una volta completato il deploy, apriamo col browser l'URL inerente al nuovo branch:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Si noti che tutti i comandi di SymfonyCloud funzionano sul branch corrente. Il deploy del branch ``sessions-in-db`` metterà a disposizione un URL simile a ``https://sessions-in-db-xxx.eu.s5y.io/``.

Testiamo il sito su questo nuovo ambiente, dovremmo vedere tutti i dati che abbiamo creato nell'ambiente master.

Se aggiungiamo altre conferenze sull'ambiente ``master``, non le visualizzeremo nell'ambiente ``sessions-in-db`` e viceversa. Questi ambienti sono indipendenti e isolati.

Se il codice di master si evolve, possiamo sempre fare un rebase del branch, risolvendo eventuali conflitti sia sul codice sia sull'infrastruttura, ed effettuare un deploy della versione aggiornata.

.. index::
    single: Symfony CLI;env:sync

Possiamo anche sincronizzare i dati dell'ambiente master con quello di ``sessions-in-db``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Effettuare il debug del deploy di produzione prima del deploy effettivo
-----------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Per impostazione predefinita, tutti gli ambienti SymfonyCloud usano le stesse impostazioni dell'ambiente ``master`` ovvero l'ambiente Symfony ``prod``. Questo ci permette di testare l'applicazione in condizioni reali. Ci dà la sensazione di sviluppare e testare direttamente sui server di produzione, ma senza i rischi associati. Questo mi ricorda i bei vecchi tempi dei deploy via FTP.

.. index::
    single: Symfony CLI;env:debug

In caso di problemi, si potrebbe voler passare all'ambiente ``dev`` di Symfony:

.. code-block:: bash

    $ symfony env:debug

Una volta che risulta tutto funzionante, torniamo alle impostazioni di produzione:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Non** abilitare mai l'ambiente ``dev`` e non abilitare mai il Profiler di Symfony sul branch ``master``; renderebbe l'applicazione davvero lenta ed esporrebbe vulnerabilità di sicurezza molto gravi.

Verifichiamo il deploy di produzione prima del vero e proprio deploy
--------------------------------------------------------------------

La possibilità di poter visionare in anteprima la prossima versione del sito web con i dati di produzione apre molte opportunità: dai test di regressione visiva ai test sulle prestazioni. `Blackfire <https://blackfire.io>`_ è lo strumento perfetto per fare questo tipo di lavoro.

Fare riferimento alla sezione sulle "Prestazioni" per saperne di più su come utilizzare Blackfire per testare il codice prima del deploy.

Merge su produzione
-------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Quando siamo soddisfatti dei cambiamenti effettuati sul branch, effettuiamo il merge del codice e dell'infrastruttura sul branch master:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-db

E ora effettuiamo il deploy:

.. code-block:: bash

    $ symfony deploy

Quando effettuiamo un deploy, solo il codice e le modifiche all'infrastruttura vengono inviate su SymfonyCloud; i dati non vengono alterati in alcun modo.

Fare pulizia
------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Infine, facciamo pulizia rimuovendo il branch e l'ambiente SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-db
    $ symfony env:delete --env=sessions-in-db --no-interaction

.. sidebar:: Andare oltre

    * `branch git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

.. _`stakeholder`: https://en.wikipedia.org/wiki/Project_stakeholder
