Adottare una metodologia
========================

Insegnare significa ripetere la stessa cosa più volte. Non lo farò, lo prometto. Alla fine di ogni passo dovresti fare un balletto e salvare il lavoro fatto fino a quel momento: è come premere ``Ctrl+S`` ma per un sito web.

Implementare una strategia con Git
----------------------------------

.. index::
    single: Git;add
    single: Git;commit

Alla fine di ogni passo non dimentichiamo di eseguire il commit con le modifiche:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Possiamo tranquillamente aggiungere "tutto", dato che Symfony gestisce un file ``.gitignore`` per noi. Inoltre ogni pacchetto può aggiungere ulteriori configurazioni. Diamo un'occhiata al contenuto attuale:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,8

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Le simpatiche stringhe sono dei segnaposto aggiunti da Symfony Flex, in modo che sappia cosa eliminare se decidessimo di rimuovere una dipendenza. Come già detto, tutto il lavoro noioso è fatto da Symfony, non da noi sviluppatori.

Potrebbe essere utile effettuare il push del nostro repository su un server da qualche parte. GitHub, GitLab o Bitbucket sono delle buone scelte.

Eseguendo il deploy su SymfonyCloud, abbiamo già una copia del repository Git, su cui però non dovremmo fare affidamento: infatti il suo scopo è solo per i deploy, non è un backup.

Continuous deploy in produzione
-------------------------------

.. index::
    single: Symfony CLI;deploy

Un'altra buona abitudine è quella di effettuare dei deploy frequentemente. Fare deploy alla fine di ogni passo è considerato una buona prassi:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
