Scoprire il cuore di Symfony
============================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

Abbiamo usato Symfony per sviluppare una potente applicazione per un bel po' di tempo, ma la maggior parte del codice eseguito dall'applicazione proviene da Symfony. Poche centinaia di righe di codice contro migliaia di righe di codice.

Mi piace capire come funzionano le cose dietro le quinte. E sono sempre stato affascinato da strumenti che mi aiutano a capire come funzionano le cose. La prima volta che ho usato un debugger passo dopo passo o la prima volta che ho scoperto ``ptrace`` sono ricordi magici.

Volete capire meglio come funziona Symfony? È tempo di scoprire come Symfony faccia funzionare un'applicazione. Invece di descrivere come Symfony gestisce una richiesta HTTP da una prospettiva teorica, che sarebbe abbastanza noiosa, useremo Blackfire per ottenere alcune rappresentazioni visive e usarlo per scoprire alcuni argomenti più avanzati.

Comprendere a fondo Symfony con Blackfire
-----------------------------------------

Sappiamo già che tutte le richieste HTTP sono servite da un unico punto di ingresso: il file ``public/index.php``. Ma cosa succede dopo? Come vengono richiamati i controller?

Profiliamo la homepage inglese in produzione con Blackfire tramite l'estensione del browser Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

O direttamente dalla linea di comando:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Andiamo alla vista "Timeline" del profilo, dovremmo vedere qualcosa di simile a quanto segue:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

Dalla timeline, passare con il mouse sulle barre colorate per avere maggiori informazioni su ogni chiamata; si imparerà molto su come funziona Symfony:

* Il punto di ingresso principale è ``public/index.php``;

* Il metodo ``Kernel::handle()`` gestisce la richiesta;

* Richiama ``HttpKernel``, che invia alcuni eventi;

* Il primo evento è ``RequestEvent``;

* Il metodo ``ControllerResolver::getController()`` determina il controller da richiamare in base all'URL;

* Il metodo ``ControllerResolver::getArguments()`` è richiamato e determina quali parametri passare al controller (usando il ParamConverter);

* Viene richiamato il metodo ``ConferenceController::index()``, in cui risiede gran parte del nostro codice;

* Il metodo ``ConferenceRepository::findAll()`` recupera tutte le conferenze dal database (si noti la connessione al database tramite ``PDO::__construct()``);

* Il metodo ``Twig\Environment::render()`` esegue il render del template;

* Sono inviati ``ResponseEvent`` e ``FinishRequestEvent``, ma sembra che nessun listener sia effettivamente registrato, in quanto sembrano essere molto veloci nell'esecuzione.

La timeline è un ottimo modo per capire come funziona un codice, che è molto utile quando si eredita un progetto sviluppato da qualcun altro.

Ora, profilare la stessa pagina dalla macchina locale nell'ambiente di sviluppo:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Apriamo il profilo. Dovremmo essere reindirizzati alla visualizzazione del grafico delle chiamate, dato che la richiesta è stata molto veloce e la timeline sarebbe stata pressoché vuota:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

È chiaro cosa sta succedendo? La cache HTTP è abilitata e quindi stiamo profilando il livello di cache HTTP di Symfony. Poiché la pagina è nella cache, ``HttpCache\Store::restoreResponse()`` sta ricevendo la risposta HTTP dalla sua cache e il controller non viene mai richiamato.

Disabilitare il livello della cache in ``public/index.php``, come abbiamo fatto nel passo precedente, e riprovare. Si vede subito che il profilo ha un aspetto molto diverso:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Le principali differenze sono le seguenti:

* ``TerminateEvent``, che non era visibile in produzione, richiede una grande percentuale del tempo di esecuzione; guardando meglio, si può vedere che questo è l'evento responsabile della memorizzazione dei dati del Profiler di Symfony raccolti durante la richiesta;

* Sotto la chiamata ``ConferenceController::index()``, si noti il metodo ``SubRequestHandler::handle()`` che esegue il render degli "Edge Side Include", d'ora in avanti ESI, (ecco perché abbiamo due chiamate a ``Profiler::saveProfile()``, una per la richiesta principale e una per l'ESI).

Esploriamo la timeline per saperne di più; passiamo alla visualizzazione del grafico delle chiamate per avere una rappresentazione diversa degli stessi dati.

Come abbiamo appena scoperto, il codice eseguito in sviluppo e in produzione è molto diverso. L'ambiente di sviluppo è più lento, dato che il Profiler di Symfony cerca di raccogliere molti dati per facilitare il debug. Per questo motivo si dovrebbe sempre profilare con l'ambiente di produzione, anche in locale.

Alcuni esperimenti interessanti: profilare una pagina di errore, profilare la pagina  ``/`` (che è un redirect) o una risorsa API. Ogni profilo vi dirà un po' di più su come funziona Symfony, quali classi e metodi sono chiamati, quali esecuzioni costano tanto e quali costano poco.

Utilizzo del plugin per il debug di Blackfire
---------------------------------------------

.. index::
    single: Blackfire;Debug Addon

Per impostazione predefinita, Blackfire rimuove tutte le chiamate non abbastanza significative, per evitare di produrre grafici troppo grandi. Quando si utilizza Blackfire come strumento di debug, è meglio tenere tutte le chiamate. L'addon "debug" ci viene in aiuto.

Dalla linea di comando, usare il parametro ``--debug``:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

In produzione, si vedrebbe ad esempio il caricamento di un file di nome ``.env.local.php``:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

Da dove viene? SymfonyCloud fa alcune ottimizzazioni quando si effettua il deploy un'applicazione Symfony, come l'ottimizzazione dell'autoloader di Composer  ( ``--optimize-autoloader --apcu-autoloader --classmap-authoritative``). Ottimizza anche le variabili d'ambiente definite nel file ``.env`` (per evitare di analizzare il file a ogni richiesta) generando il file ``.env.local.php``:

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire è uno strumento molto potente, che aiuta a capire il modo in cui il codice viene eseguito da PHP. Migliorare le prestazioni è solo uno dei modi per utilizzare un profiler.
