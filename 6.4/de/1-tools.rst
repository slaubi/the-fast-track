Prüfe Deine Arbeitsumgebung
============================

Bevor wir mit der Arbeit an dem Projekt beginnen, müssen wir sicherstellen, dass Du eine gute Arbeitsumgebung hast – das ist sehr wichtig. Die Entwicklungswerkzeuge, die uns heute zur Verfügung stehen, unterscheiden sich stark von denen, die wir vor 10 Jahren hatten. Sie haben sich sehr weiterentwickelt, zum Besseren. Es wäre eine Schande, sie nicht zu nutzen. Gute Werkzeuge können Dich weit bringen.

Bitte überspringe diesen Schritt nicht, oder lies zumindest den letzten Abschnitt über die Symfony CLI.

Ein Computer
------------

Du brauchst einen Computer. Die gute Nachricht ist, dass es auf jedem gängigen Betriebssystem läuft: macOS, Windows oder Linux. Symfony und alle Tools, die wir verwenden werden, sind mit allen Dreien kompatibel.

Meine eigenen Entscheidungen
----------------------------

Ich möchte effizient mit den besten der verfügbaren Möglichkeiten arbeiten. Ich habe deswegen Entscheidungen für dieses Buch getroffen, die auf meinen persönlichen Erfahrungen basieren.

`PostgreSQL`_ wird unsere Wahl für Alles sein: von der Datenbank zur Queue, über Caching bis zur Sessionspeicherung. Für die meisten Projekte ist PostgreSQL die beste Lösung. Es verfügt über eine gute Skalierbarkeit und vereinfacht die Infrastruktur, in dem es nur einen Dienst zu managen gibt.

Am Endes des Buches werden wir lernen wie man `RabbitMQ`_ für Warteschlangen (queues) und `Redis`_ für Sessions verwendet.

IDE
---

.. index:: IDE

Du könntest Notepad verwenden, wenn Du möchtest. Ich würde es aber nicht empfehlen.

Früher habe ich mit Textmate gearbeitet. Der Komfort bei der Verwendung einer echten IDE ist unbezahlbar. Automatische Vervollständigung, das Hinzufügen und Sortieren von ``use``-Anweisungen und das Springen von einer Datei zur anderen sind nur einige Funktionen, die Deine Produktivität steigern werden.

Ich würde empfehlen, `Visual Studio Code`_ oder `PhpStorm`_ zu verwenden. Erstere ist kostenlos, letztere nicht, hat aber eine bessere Symfony-Integration (dank des `Symfony Support Plugins`_). Die Entscheidung liegt ganz bei Dir. Ich weiß, dass Du gerne wissen möchtest, welche IDE ich verwende. Ich verwende für dieses Buch Visual Studio Code.

Terminal
--------

.. index:: Terminal

Wir werden immer wieder zwischen der IDE und der Kommandozeile wechseln. Du könnest auch das eingebaute Terminal Deiner IDE verwenden, ich persönlich bevorzuge aber ein echtes, um mehr Platz zu haben.

Linux kommt mit dem eingebauten ``Terminal``, verwende `iTerm2`_ unter macOS und unter Windows funktioniert `Hyper`_ gut.

Git
---

.. index:: Git

Für die Versionskontrolle werden wir `Git`_ benutzen, da es derzeit von allen benutzt wird.

Installiere `Git bash`_ unter Windows.

Stelle sicher, dass Du weißt, wie man übliche Operationen wie beispielsweise ``git clone``, ``git log``, ``git show``, ``git diff`` oder ``git checkout`` ausführt.

PHP
---

.. index::
    single: PHP
    single: PHP extensions

Wir werden Docker für Services verwenden, aber ich möchte PHP aus Performance-, Stabilitäts- und Komplexitätsgründen auf meinem lokalen Computer installiert haben. Nenn' mich altmodisch, wenn Du willst, aber die Kombination aus lokalem PHP und Docker Services ist die perfekte Kombination für mich.

Verwende PHP 8.3 und überprüfe, ob die folgenden PHP-Erweiterungen installiert sind oder installiere sie jetzt: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Optional installiere auch ``redis``, ``curl`` and  ``zip``.

Du kannst alle aktivierten Erweiterungen mit ``php -m`` überprüfen.

Außerdem brauchen wir ``php-fpm``, wenn Deine Plattform es unterstützt. ``php-cgi`` funktioniert auch.

Composer
--------

.. index:: Composer

Die Verwaltung von Abhängigkeiten (dependencies) ist bei einem Symfony-Projekt heutzutage alles. Hole Dir die neueste Version von `Composer`_, dem Paketverwaltungswerkzeug für PHP.

Wenn Du mit Composer nicht vertraut bist, nimm Dir etwas Zeit, um Dich einzulesen.

.. tip::

    Du musst nicht die vollständigen Befehlsnamen eingeben: ``composer req`` macht das gleiche wie ``composer require``, verwende ``composer rem`` statt ``composer remove``, ...

NodeJS
------

Wir werden nicht viel Javascript Code schreiben, aber wir werden JavaScript/NodeJS tools nutzen um unsere Assets zu verwalten. Stelle sicher, dass Du `NodeJS`_ installiert hast.

Docker und Docker Compose
-------------------------

.. index:: Docker,Docker Compose

Die Services werden von Docker und Docker Compose verwaltet. `Installiere sie`_ und starte Docker. Wenn Du Docker zum ersten Mal benutzt, mache Dich mit dem Tool vertraut. Aber keine Panik, unsere Anwendung wird sehr einfach sein. Keine ausgefallenen Konfigurationen, kein komplexes Setup.

Symfony CLI
-----------

.. index:: Symfony CLI

Zu guter Letzt werden wir mit der ``symfony`` CLI unsere Produktivität steigern. Sie stellt einen lokalen Webserver bereit, bietet vollständige Docker-Integration und Cloud-Unterstützung durch Platform.sh an. Dadurch werden wir viel Zeit sparen.

Installiere die `Symfony CLI`_ jetzt.

Um HTTPS lokal nutzen zu können, müssen wir auch `eine Zertifizierungsstelle (certificate authority -CA) installieren`_, um den TLS-Support zu aktivieren. Führe den folgenden Befehl aus:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: terminal
    :class: ignore

    $ symfony server:ca:install

Überprüfe, ob Dein Computer alle erforderlichen Anforderungen erfüllt, indem Du den folgenden Befehl ausführst:

.. code-block:: terminal
    :class: ignore

    $ symfony book:check-requirements

Wenn Du Lust auf mehr hast, kannst Du auch den `Symfony-Proxy`_ nutzen. Er ist optional, aber er erlaubt Dir, für Dein Projekt einen lokalen Domainnamen zu haben, der mit ``.wip`` endet.

Wenn wir einen Befehl in einem Terminal ausführen, werden wir ihm fast immer das Präfix ``symfony`` geben, wie ``symfony composer`` anstelle von lediglich ``composer``, oder ``symfony console`` anstelle von ``./bin/console``.

Der Hauptgrund dafür ist, dass die Symfony CLI automatisch einige Environment-Variablen (Umgebungsvariablen) setzt, passend zu den Diensten, die auf Deinem Computer über Docker laufen. Diese Environment-Variablen stehen für HTTP-Requests zur Verfügung, da sie vom lokalen Webserver automatisch gesetzt werden. Die Verwendung von ``symfony`` auf dem CLI stellt also sicher, dass Du in jedem Kontext das gleiche Verhalten hast.

Darüber hinaus wählt die Symfony CLI automatisch die bestmögliche PHP-Version für das Projekt aus.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`NodeJS`: https://nodejs.org/
.. _`Installiere sie`: https://docs.docker.com/install/
.. _`Symfony CLI`: https://symfony.com/download
.. _`eine Zertifizierungsstelle (certificate authority -CA) installieren`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`Symfony Support Plugins`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`Symfony-Proxy`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
