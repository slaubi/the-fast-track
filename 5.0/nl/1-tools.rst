Jouw werkomgeving controleren
=============================

Voordat we aan het project beginnen is het belangrijk dat we controleren of iedereen een goede werkomgeving heeft. De beschikbare ontwikkeltools zijn de laatste jaren sterk geëvolueerd en zien er vandaag de dag heel anders uit dan 10 jaar geleden. Het zou zonde zijn om dit niet ten volle te benutten. Goede tools kunnen je een heel eind op weg helpen.

Sla deze stap alsjeblieft niet over. Of lees in ieder geval het laatste deel over de Symfony CLI.

Een computer
------------

Je hebt een computer nodig. Het goede nieuws is dat elk populair OS gebruikt kan worden: macOS, Windows of Linux. Symfony, en alle hulpmiddelen die we gaan gebruiken, zijn hier allemaal compatibel mee.

Uitgesproken keuzes
-------------------

Ik wil snel handelen met de beste opties die er zijn. Ik heb voor dit boek uitgesproken keuzes gemaakt.

`PostgreSQL`_ wordt onze keuze voor de database engine.

`RabbitMQ`_ is de winnaar voor de queues.

IDE
---

.. index:: IDE

Je kan Notepad gebruiken als je dat wilt. Ik zou het echter niet aanraden.

Ik werkte vroeger met Textmate. Nu niet meer. Het comfort van het gebruik van een "echte" IDE is van onschatbare waarde. Automatisch code aanvullen, ``use`` statements toevoegen, het automatisch sorteren  daarvan en van het ene bestand naar het andere springen zijn een paar functies die je productiviteit zullen verhogen.

Ik zou aanraden om `Visual Studio Code`_ of `PhpStorm`_ te gebruiken. De eerste is gratis, de tweede niet, maar PhpStorm heeft een betere integratie met Symfony (dankzij de `Symfony Support Plugin`_ ). Het is uiteindelijk aan jou. Je wilt vast weten welke IDE ik gebruik. Ik schrijf dit boek in Visual Studio Code.

Terminal
--------

.. index:: Terminal

We zullen de hele tijd van de IDE naar de command line overschakelen. Je kan de ingebouwde terminal van jouw IDE gebruiken, maar ik geef de voorkeur aan een echte terminal om meer ruimte te hebben.

Linux wordt geleverd met ingebouwde ``Terminal``. Gebruik `iTerm2`_ op macOS. Op Windows werkt `Hyper`_ goed.

Git
---

.. index:: Git

Mijn laatste boek raadde Subversion aan voor versiebeheer. Het lijkt erop dat iedereen nu `Git`_ gebruikt.

Op Windows, installeer `Git bash`_.

Zorg ervoor dat je weet hoe je de alledaagse operaties zoals ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout`` moet uitvoeren.

PHP
---

.. index:: PHP

We zullen Docker gebruiken voor services, maar ik heb PHP graag op mijn lokale computer geïnstalleerd vanwege snelheid, stabiliteit en eenvoud. Het kan oubollig overkomen, maar de combinatie van een lokale PHP-installatie en Docker services werkt voor mij perfect.

Gebruik minstens PHP 7.3, indien mogelijk 7.4. Controleer of de :index:`following PHP extensions <PHP extensions>` geïnstalleerd zijn, of installeer ze nu: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl`` en ``sodium``. Installeer optioneel ook de ``redis`` en ``curl`` extensies.

Je kunt controleren welke extensies momenteel ingeschakeld zijn door ``php -m`` uit te voeren.

We ook hebben ``php-fpm`` nodig als jouw platform dit ondersteunt, maar ``php-cgi`` werkt net zo goed.

Composer
--------

.. index:: Composer

Het beheren van dependencies is tegenwoordig essentieel voor een Symfony-project. Download de nieuwste versie van de package management tool voor PHP: `Composer <https://getcomposer.org/>`_.

Als je niet bekend bent met Composer, neem dan even de tijd om erover te lezen.

.. tip::

    Je hoeft niet de volledige commando's in te voeren. ``composer req`` doet hetzelfde als ``composer require`` en ``composer rem`` doet bijvoorbeeld hetzelfde als ``composer remove``, ...

Docker en Docker Compose
------------------------

.. index:: Docker,Docker Compose

De services worden beheerd door Docker en Docker Compose. `Installeer ze <https://docs.docker.com/install/>`_ en start Docker. Als je voor het eerst gebruik maakt van deze tool, neem dan eerst de tijd om er bekend mee te raken. Maar geen paniek, we gaan er niks ingewikkelds mee doen. Geen buitensporige configuraties, geen complexe instellingen.

Symfony CLI
-----------

.. index:: Symfony CLI

Tenslotte zullen we de ``symfony`` CLI gebruiken om onze productiviteit te verhogen. Van de lokale webserver, tot volledige Docker-integratie en SymfonyCloud-ondersteuning; het zal ons veel tijd besparen.

Installeer de `Symfony CLI <https://symfony.com/download>`_ en verplaats deze naar jouw ``$PATH``. Maak een `SymfonyConnect <https://connect.symfony.com/>`_ account  aan als je die nog niet hebt en log in via ``symfony login``.

Om lokaal HTTPS te kunnen gebruiken, moeten we ook `een CA installeren <https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls>`_ voor TLS ondersteuning. Voer het volgende commando uit:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Controleer of jouw computer aan alle vereisten voldoet door het volgende commando uit te voeren:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

Als je het helemaal mooi wil doen, dan kan je ook de `Symfony-proxy <https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy>`_ gebruiken. Dit is optioneel, maar het zorgt ervoor dat je een lokale domeinnaam met de extensie ``.wip`` kan gebruiken voor je project.

Bij het uitvoeren van een commando in een terminal, zullen we er bijna altijd ``symfony`` voor zetten, als in ``symfony composer`` in plaats van gewoon ``composer`` , of ``symfony console`` in plaats van ``./bin/console`` .

De belangrijkste reden daarvoor is dat de Symfony CLI automatisch enkele omgevingsvariabelen instelt op basis van de services die via Docker op jouw machine draaien. Deze omgevingsvariabelen zijn beschikbaar voor HTTP requests omdat de lokale webserver ze automatisch injecteert. Het gebruik van ``symfony`` op de CLI zorgt er dus voor dat je over de gehele omgeving hetzelfde gedrag hebt.

Bovendien selecteert de Symfony CLI automatisch de "best mogelijke" PHP-versie voor het project.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
