Preparazione dell'ambiente di lavoro
====================================

Prima di iniziare a lavorare al progetto, è molto importante assicurarci che tutti abbiano a disposizione un buon ambiente di lavoro. Gli strumenti di cui disponiamo oggi sono molto diversi da quelli che avevamo dieci anni fa. Tali strumenti si sono evoluti molto e sarebbe un peccato non sfruttarli adeguatamente, visto che possono fare la differenza.

È bene non saltare questo passaggio. Consiglio almeno di leggere l'ultima sezione sulla CLI di Symfony.

Un computer
-----------

Avrete bisogno di un computer. La buona notizia è che il progetto può essere eseguito su uno qualsiasi fra i sistemi operativi più comuni, come macOS, Windows o Linux. Symfony e tutti gli strumenti che useremo sono compatibili con ognuna di queste piattaforme.

Preferenze personali
--------------------

Voglio poter lavorare velocemente usando gli strumenti migliori fra quelli disponibili, perciò ho fatto scelte in base alle mie preferenze personali per questo libro.

`PostgreSQL`_ sarà la nostra scelta per gestire: database, code di messaggi, la cache e il salvataggio delle sessioni. Nella maggior parte dei progetti PostgreSQL è la soluzione migliore, in quanto offre una scalabilità migliore e permette di semplificare l'infrastruttura avendo un solo servizio da gestire.

Alla fine del libro, impareremo ad utilizzare `RabbitMQ`_ per le code di messaggi e `Redis`_ per le sessioni.

IDE
---

.. index:: IDE

È possibile utilizzare Notepad se lo si desidera, ma è un'opzione che non raccomando.

In passato utilizzavo Textmate, ma la comodità e le funzionalità di un vero IDE mi hanno fatto cambiare idea. Il completamento automatico, le istruzioni ``use`` aggiunte e ordinate automaticamente, e poter saltare con facilità da un file all'altro sono solo alcune delle caratteristiche che aumentano immediatamente la produttività.

Consiglierei di usare `Visual Studio Code`_ o `PhpStorm`_. Il primo è gratuito, mentre il secondo non lo è, ma ha una migliore integrazione con Symfony (grazie al `plugin per il supporto di Symfony`_). Ognuno è libero di scegliere quale utilizzare. Per chi si stesse chiedendo quale IDE utilizzi io stesso, sto scrivendo questo libro con Visual Studio Code.

Terminale
---------

.. index:: Terminal

Alterneremo l'uso di un IDE e linea di comando. Nonostante gli IDE abbiano spesso dei terminali integrati, personalmente preferisco usarne uno dedicato in modo da poter avere più spazio a disposizione.

Su Linux si può trovare ``Terminale`` preinstallato, mentre su macOS si può utilizzare `iTerm2`_. Su Windows, `Hyper`_ può essere una buona scelta.

Git
---

.. index:: Git

Ai tempi del mio ultimo libro, Subversion era lo strumento più diffuso per il controllo delle versioni del codice. Ora la maggioranza degli sviluppatori usa `Git`_.

Su Windows, installiamo `Git bash`_.

Assicuriamoci di conoscere i comandi più comuni, come ad esempio ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``, ...

PHP
---

.. index:: PHP

Nonostante l'utilizzo di Docker, mi piace in ogni caso avere PHP installato localmente sul mio computer per motivi di prestazioni, stabilità, e semplicità. Forse sarò all'antica, ma trovo perfetta la combinazione di un ambiente PHP locale e delle funzionalità di Docker.

Useremo PHP 8.0, controllate che queste :index:`estensioni PHP <PHP extensions>` siano installate: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Le estensioni ``redis``, ``curl``, e ``zip`` sono consigliate, ma opzionali.

È possibile controllare le estensioni attualmente abilitate eseguendo il comando ``php -m``.

Se supportato dalla piattaforma in uso useremo ``php-fpm``, ma come alternativa ``php-cgi`` funziona comunque bene.

Composer
--------

.. index:: Composer

Al giorno d'oggi uno strumento per gestire le dipendenze è fondamentale. Installeremo l'ultima versione di `Composer`_, lo strumento di gestione dei pacchetti per PHP.

Se non si ha familiarità con Composer, è il caso di fermarsi un attimo per capirne il funzionamento.

.. tip::

    Non è necessario digitare i nomi completi dei comandi: ``composer req`` funziona come ``composer require`` , così come ``composer rem`` è equivalente a ``composer remove``, e così via.

NodeJS
------

Non scriveremo molto codice JavaScript, ma useremo strumenti JavaScript/NodeJS per gestire i nostri asset. Verificare di avere installato `NodeJS`_ e il gestore di pacchetti `Yarn`_.

Docker e Docker Compose
-----------------------

.. index:: Docker,Docker Compose

I servizi saranno gestiti da Docker e Docker Compose. Bisogna `installarli`_ e avviare Docker. Se si è alle prime armi, passare qualche minuto ad acquisire familiarità con lo strumento è consigliato. Ma il nostro uso sarà molto semplice, per cui non è il caso di spaventarsi: le configurazioni che useremo saranno lineari e razionali.

Symfony CLI
-----------

.. index:: Symfony CLI

Per ultima, ma non per questo meno importante, useremo la CLI ``symfony``. Grazie al suo server locale, alla piena integrazione con Docker, e al supporto di SymfonyCloud, ci permetterà di risparmiare molto tempo e di conseguenza essere più produttivi.

Installiamo il `binario di Symfony (Symfony CLI)`_  e spostiamolo all'interno del proprio ``$PATH``. Se non se ne dispone già, creiamo un account `SymfonyConnect`_  ed effettuiamo infine il login tramite il comando ``symfony login``.

Per poter utilizzare HTTPS localmente, occorre installare `un'autorità di certificazione (CA)`_ per abilitare il supporto a TLS. Eseguiamo il seguente comando:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Verifichiamo che il computer abbia tutti i requisiti necessari eseguendo il seguente comando:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

Come tocco finale, si può anche eseguire il `proxy di Symfony`_. È opzionale, ma ci permette di utilizzare un nome di dominio locale con suffisso ``.wip`` per il nostro progetto.

Quando eseguiremo un comando in un terminale, lo faremo iniziare quasi sempre con ``symfony``. Ad esempio useremo ``symfony composer`` invece di un semplice ``composer``, o ``symfony console`` posto di ``./bin/console``.

La ragione principale è che la CLI di Symfony imposta automaticamente alcune variabili d'ambiente in base ai servizi in esecuzione tramite Docker. Queste variabili d'ambiente sono disponibili per le richieste HTTP poiché il server web locale le aggiunge automaticamente. Per questo motivo, l'utilizzo della CLI ``symfony`` ci garantisce lo stesso comportamento su tutti gli ambienti.

Inoltre, la CLI di Symfony seleziona automaticamente la versione "migliore" di PHP per il progetto.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`NodeJS`: https://nodejs.org/
.. _`Yarn`: https://classic.yarnpkg.com/en/docs/install/
.. _`installarli`: https://docs.docker.com/install/
.. _`binario di Symfony (Symfony CLI)`: https://symfony.com/download
.. _`SymfonyConnect`: https://symfony.com/connect/login
.. _`un'autorità di certificazione (CA)`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`plugin per il supporto di Symfony`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`proxy di Symfony`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
