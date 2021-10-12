Presentazione del progetto
==========================

Dobbiamo trovare un progetto su cui lavorare: uno sufficientemente grande da coprire tutti gli aspetti di Symfony, ma allo stesso tempo abbastanza piccolo. Non è il caso di annoiarsi implementando funzionalità noiose e ripetitive.

Sveliamo il progetto
--------------------

Poiché questo libro deve essere rilasciato durante la SymfonyCon ad Amsterdam, potrebbe essere utile se il progetto fosse in qualche modo legato al mondo di Symfony e alle conferenze. Per questo ho pensato a un `libro degli ospiti <https://en.wikipedia.org/wiki/Guestbook>`_, trovo bella l'idea di fare un salto nel passato e sviluppare un libro degli ospiti nel 2019!

Ci siamo: il progetto serve per poter ricevere un feedback alle conferenze. In homepage ci sarà un elenco delle conferenze, e ognuna avrà la propria pagina con dei commenti. Ogni commento sarà composto da un breve testo e (facoltativamente) da una foto scattata durante la conferenza. Credo che questi siano tutti i requisiti necessari per poter iniziare.

Il *progetto* conterrà diverse *applicazioni* . Un'applicazione web tradizionale con un frontend HTML, una _API_ e una _SPA_ per cellulari. Può andare bene?

Imparare è fare
---------------

Per imparare davvero qualcosa, bisogna farla. Leggere un libro su Symfony può essere bello, ma scrivere il codice al computer mentre si legge il libro è ancora meglio. Nello scrivere questo libro mi sono assicurato di fare tutto il necessario per permettere di seguire e ottenere gli stessi risultati che ho avuto quando ho creato l'applicazione inizialmente.

Il libro contiene tutto il codice da scrivere e tutti i comandi da eseguire per ottenere il risultato finale. Non manca nulla. Questo è possibile perché le moderne applicazioni Symfony hanno pochissimo codice _boilerplate_, e la maggior parte del codice che scriveremo insieme riguarda la *logica di business* del progetto. Tutto il resto è per lo più automatizzato o generato automaticamente per noi.

Il diagramma finale dell'infrastruttura
---------------------------------------

Anche se l'idea di base sembra semplice, non costruiremo un progetto stile "Hello World". Non useremo solo PHP e un database.

L'obiettivo è quello di creare un progetto con alcune delle complessità che si possono trovare nel lavoro di tutti i giorni. Basta dare un'occhiata all'infrastruttura finale del progetto:

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

Uno dei grandi vantaggi dell'uso di un framework è la ridotta quantità di codice necessario per sviluppare un progetto di questo tipo:

* 20 classi PHP all'interno di ``src/``;

* 550 righe di codice contenente logica (_LLOC_) come riportato da `PHPLOC <https://github.com/sebastianbergmann/phploc>`_;

* 40 righe di configurazione in tre file (tramite annotazioni e YAML), necessarie principalmente alla configurazione del design per il backend;

* 20 righe di configurazione dell'infrastruttura di sviluppo (Docker);

* 100 righe di configurazione dell'infrastruttura di produzione (SymfonyCloud);

* 5 variabili d'ambiente esplicite.

Pronti per la sfida?

Ottenere il codice sorgente del progetto
----------------------------------------

Per continuare con un approccio retrò, avrei potuto creare un CD contenente il codice sorgente, giusto? Ma forse è meglio un repository Git?

.. index::
    single: Project;Git Repository
    single: Git;clone

Cloniamo il `repository del libro degli ospiti <https://github.com/the-fast-track/book-5.0-1>`_ da qualche parte nella nostra macchina locale:

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.0-1 --book guestbook

Questo repository contiene tutto il codice del libro.

Si noti che stiamo usando ``symfony new`` invece di ``git clone``, in quanto questo non si limita a clonare il repository (ospitato su Github sotto l'organizzazione ``the-fast-track``: ``https://github.com/the-fast-track/book-5.0-1``), ma avvia anche il server web, i container, migra il database, carica le fixture... Dopo aver eseguito il comando, l'applicazione dovrebbe essere attiva e funzionante, pronta per essere utilizzata.

Posso garantire che il codice nel libro sia al 100% sincronizzato con quello nel repository (utilizzando l'URL esatto fornito precedentemente). Sincronizzare manualmente le modifiche fra libro e codice sorgente nel repository è quasi impossibile. Parlo per esperienza, in passato ci ho provato e ho fallito: è semplicemente impossibile, in particolar modo per la tipologia di libri che scrivo io, quelli in cui si parla di sviluppo di applicazioni web. Poiché ogni capitolo dipende da quelli precedenti, ogni modifica potrebbe avere conseguenze su tutti i capitoli successivi.

La buona notizia è che il repository Git per questo libro è *generato automaticamente* dal contenuto del libro. Proprio così, mi piace automatizzare tutto, perciò ho creato uno script che ha il compito di leggere il libro e creare il repository Git. C'è un bell'effetto collaterale: qualora il libro fosse modificato, lo script fallirà in caso di modifiche incoerenti o se mi dovessi dimenticare di aggiornare alcune istruzioni. Questo è BDD, *Book Driven Development*!

Esplorando il codice sorgente
-----------------------------

Ancora meglio, il repository non contiene solo la versione finale del codice sul branch ``master``. Lo script esegue ogni azione illustrata nel libro, e fa un _commit_ alla fine di ogni sezione. Crea anche dei tag a ogni passo per facilitare l'esplorazione del codice. Bello, vero?

.. index::
    single: Git;checkout

Se si è pigri, si può visualizzare lo stato del codice alla fine di ogni passo facendo il checkout del tag appropriato. Ad esempio, se vogliamo leggere e testare il codice alla fine del passo 10, possiamo eseguire questo comando:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Così come per la clonazione del repository, non stiamo usando ``git checkout``, ma ``symfony book:checkout``. Il comando assicura che, qualunque sia lo stato in cui ci si trova attualmente, si possa avere un'applicazione funzionante per il passo che si richiede. **Attenzione: tutto il codice, dati e container sono sovrascritti con questa operazione.**

È inoltre possibile controllare qualsiasi passo intermedio:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

Come ho già detto, consiglio di scrivere codice da soli, ma se si dovesse rimanere bloccati, si può confrontare il proprio progetto con quello del libro.

.. index::
    single: Git;diff

Non si è sicuri di aver fatto tutto correttamente in 10.2? Basta controllare le differenze nel codice:

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Si vuole sapere quando un file è stato creato o modificato?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

Si possono anche visualizzare diff, tag e commit direttamente su GitHub. Questo è un ottimo modo per copiare il codice se si sta leggendo un libro cartaceo!
