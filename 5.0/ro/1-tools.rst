Pregătirea mediului de lucru
=============================

Înainte de a începe să lucrăm la proiect, trebuie să ne pregătim un mediu de lucru optim. Este foarte important. Instrumentele de dezvoltare pe care le avem astăzi la dispoziție sunt foarte diferite de cele pe care le-am avut acum 10 ani. Au evoluat mult și ar fi păcat să nu le folosim. Instrumentele bune te pot avantaja enorm.

Nu neglija acest pas. Sau cel puțin, citește ultima secțiune despre Symfony CLI.

Un computer
-----------

Ai nevoie de un computer. Vestea bună este că poate rula pe orice sistem de operare popular: macOS, Windows sau Linux. Symfony și toate instrumentele pe care le vom folosi sunt compatibile cu fiecare dintre acestea.

Alegeri personale
-----------------

Vreau să mă mișc rapid cu cele mai bune opțiuni existente. Am folosit alegeri personale pentru această carte.

`PostgreSQL`_ va fi alegerea noastră pentru baza de date.

`RabbitMQ`_ este câștigătorul la capitolul cozi de mesaje.

IDE
---

.. index:: IDE

Poți utiliza Notepad, dacă dorești. Dar eu nu l-aș recomanda.

Obișnuiam să lucrez cu Textmate, dar nu-l mai folosesc. Confortul utilizării unui IDE „real“ nu are preț. Completarea automată, declarațiile ``use`` adăugate și sortate automat, saltul dintr-un fișier în altul sunt câteva caracteristici ce îți pot spori productivitatea.

Ți-aș recomanda să folosești `Visual Studio Code`_ sau `PhpStorm`_. Primul este gratuit, cel de-al doilea nu, dar are o mai bună integrare cu Symfony (datorită `modulului Symfony Support`_). Depinde de preferințele tale. Probabil vrei să știi ce IDE folosesc eu. Scriu această carte în Visual Studio Code.

Consola
-------

.. index:: Terminal

Vom trece de la IDE la linia de comandă frecvent. Poți utiliza terminalul integrat al IDE-ului tău, dar prefer să folosesc unul dedicat pentru a avea mai mult spațiu.

Linux are o consolă incorporată denumită ``Terminal``. Pe macOS poți folosi `iTerm2`_. Pe Windows funcționează bine `Hyper`_.

Git
---

.. index:: Git

În ultima mea carte am recomandat Subversion pentru versionare. Se pare că acum toată lumea folosește `Git`_.

Pe Windows instalează `Git bash`_.

Asigură-te că ești familiar cu operațiunile comune, cum ar fi ``git clone``, ``git log``, ``git show``, ``git dif``, ``git checkout``, ...

PHP
---

.. index:: PHP

Vom folosi Docker pentru servicii, dar îmi place să am PHP instalat pe computerul meu local din motive de performanță, stabilitate și simplitate. Poate sunt de modă veche, dacă vrei, dar combinația dintre serviciile locale PHP și Docker este perfectă pentru mine.

Folosește PHP 7.3 dacă poți, sau chiar 7.4, în funcție de când citești această carte. Verifică ca  :index:`următoarele extensii PHP <PHP extensions>` sunt instalate sau instalează-le acum: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Opțional instalează ``redis`` și ``curl``.

Poți verifica extensiile activate rulând ``php -m`` în linia de comandă.

Vom avea de asemenea nevoie și de ``php-fpm``, dacă platforma ta suportă asta, dar funcționează și ``php-cgi``.

Composer
--------

.. index:: Composer

În zilele noastre, gestiunea dependințelor este foarte importantă pentru un proiect Symfony. Descarcă cea mai recentă versiune `Composer <https://getcomposer.org/>`_, managerul de pachete și librării pentru PHP.

Dacă nu ești familiarizat cu Composer, rezervă-ți puțin timp pentru a te documenta.

.. tip::

    Nu o să avem nevoie de numele întreg al comenzilor: ``composer req`` funcționează la fel ca ``composer require``, sau ``composer rem`` în loc de ``composer remove``, ...

Docker și Docker Compose
-------------------------

.. index:: Docker,Docker Compose

Serviciile vor fi gestionate de Docker și Docker Compose. Instalează-le `<https://docs.docker.com/install/>`_  și pornește Docker. Dacă utilizezi Docker pentru prima dată, familiarizează-te cu el. Totuși nu te panica, vom avea o configurare foarte simplă. Fără configurații fanteziste sau complexe.

Symfony CLI
-----------

.. index:: Symfony CLI

Nu în ultimul rând, vom folosi utilitarul ``symfony`` din linia de comandă pentru a ne spori productivitatea. De la serverul web local pe care îl oferă, până la integrarea completă cu Docker și suportul pentru SymfonyCloud, ne va economisi mult timp.

Instalează `Symfony CLI <https://symfony.com/download>`_ și mută-l sub calea ``$PATH``. Creează un cont `SymfonyConnect <https://connect.symfony.com/>`_, dacă nu ai deja unul, și conectează-te prin ``symfony login``.

Pentru a utiliza HTTPS local trebuie de asemenea să `instalăm o autoritate de certificare <https://symfony.com/doc/current/setup/symfony_server.html#enablement-tls>`_ pentru a activa suportul TLS. Execută următoarea comandă:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Asigură-te că sistemul tău îndeplinește cerințele necesare executând următoarea comandă:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

De asemenea dacă vrei, poți folosi `Symfony proxy <https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy>`_. Este opțional, dar îți permite să obții un nume de domeniu local pentru proiectul tău, care se încheie cu ``.wip``.

Atunci când executăm o comandă în linia de comandă, aproape întotdeauna o vom prefixa cu ``symfony``, de exemplu: ``symfony composer`` în loc de doar ``composer``, sau ``symfony console`` în loc de ``./bin/console``.

Motivul principal este că Symfony CLI stabilește automat unele variabile de mediu bazate pe serviciile care rulează pe sistemul tău prin Docker. Aceste variabile de mediu sunt disponibile pentru solicitările HTTP deoarece serverul web local le injectează automat. Așadar, folosind ``symfony`` în linia de comandă asigură același comportament pe toate instanțele.

Mai mult, Symfony CLI selectează automat cea mai bună versiune PHP posibilă pentru proiect.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`modulului Symfony Support`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
