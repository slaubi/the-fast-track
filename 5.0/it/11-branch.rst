Creare branch del codice
========================

Esistono molti modi per organizzare il flusso di lavoro per modificare il codice di un progetto, ma lavorare sul branch master ed effettuare un deploy direttamente in produzione del codice senza una suite di test, probabilmente, non è la scelta migliore.

Testare l'applicazione non riguarda solo i test unitari o funzionali, ma anche il controllo del comportamento dell'applicazione con i dati di produzione. Se voi o i vostri `stakeholder`_ potete utilizzare l'applicazione esattamente come sarà distribuita agli utenti finali, questo sarà un enorme vantaggio e vi permetterà di effettuare deploy con più sicurezza. Mettere in condizione delle persone non tecniche di validare le nuove funzionalità è un concetto importantissimo.

Per semplicità e per evitare di ripeterci continueremo a lavorare nel branch master, ma più avanti vedremo come migliorare questo aspetto.

Adottare un flusso di lavoro con git
------------------------------------

Un possibile flusso di lavoro potrebbe essere quello di creare un branch per ogni nuova funzionalità o la risoluzione di un bug. Quest'approccio è semplice ed efficiente.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Creazione di branch
-------------------

.. index::
    single: Git;branch
    single: Git;checkout

Il flusso di lavoro inizia con la creazione di un nuovo branch git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Questo comando crea un nuovo branch dal nome ``sessions-in-redis`` partendo dal branch ``master``. Sarà una "biforcazione" del codice e della configurazione dell'infrastruttura.

Salvare le sessioni su Redis
----------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Come probabilmente avrete intuito dal nome del branch, vogliamo cambiare il salvataggio delle sessioni, togliendole dal filesystem per memorizzarle su Redis.

I passi necessari per fare questo sono semplicemente:

#. Creare un branch git;

#. Aggiornare la configurazione di Symfony, se necessario;

#. Aggiungere e/o modificare qualche linea di codice se necessario;

#. Aggiornare la configurazione di PHP (aggiungendo l'estensione Redis per PHP);

#. Aggiornare l'infrastruttura su Docker e SymfonyCloud (aggiungendo il servizio Redis);

#. Testare nel nostro ambiente in locale;

#. Effettuare dei test anche da remoto;

#. Effettuare un merge del branch su master;

#. Deploy in produzione;

#. Cancellare il branch.

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

Guardiamo il sito locale. Poiché non ci sono cambiamenti visivi e poiché non stiamo ancora usando le sessioni, dovrebbe funzionare tutto come prima.

Deploy di un branch
-------------------

.. index::
    single: SymfonyCloud;Environment

Prima di passare al deploy in produzione, dovremmo testare il branch su un'infrastruttura identica a quella di produzione. Dovremmo anche controllare che tutto funzioni bene sull'ambiente Symfony ``prod`` (il sito locale usava l'ambiente Symfony ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Ora, creiamo un *ambiente SymfonyCloud* a partire dal *branch*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Con il seguente comando creeremo un nuovo ambiente:

* Il branch eredita il codice e l'infrastruttura dall'attuale branch (``sessions-in-redis``);

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

Si noti che tutti i comandi di SymfonyCloud funzionano sul branch corrente. Il deploy del branch ``sessions-in-redis`` metterà a disposizione un URL simile a ``https://sessions-in-redis-xxx.eu.s5y.io/``, che si potrà aprire con questo comando.

Testiamo il sito su questo nuovo ambiente, dovremmo vedere tutti i dati che abbiamo creato nell'ambiente master.

Se aggiungiamo altre conferenze sull'ambiente ``master``, non le visualizzeremo nell'ambiente ``sessions-in-redis`` e viceversa. Gli ambienti sono indipendenti e isolati.

Se il codice di master si evolve, possiamo sempre fare un rebase del branch, risolvendo eventuali conflitti sia sul codice sia sull'infrastruttura, ed effettuare un deploy della versione aggiornata.

.. index::
    single: Symfony CLI;env:sync

Possiamo anche sincronizzare i dati dell'ambiente master con quello di ``sessions-in-redis``:

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
    $ git merge sessions-in-redis

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

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Andare oltre

    * `branch git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`stakeholder`: https://en.wikipedia.org/wiki/Project_stakeholder
