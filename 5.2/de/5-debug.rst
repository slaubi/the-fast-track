Fehlerbehebung
==============

Bei der Einrichtung eines Projekts geht es auch darum, die richtigen Werkzeuge zu haben, um Probleme zu beheben.

Weitere Abhängigkeiten installieren
-----------------------------------

Denke daran, dass das Projekt mit sehr wenigen Abhängigkeiten (Dependencies) erstellt wurde. Keine Template-Engine, keine Debug-Tools, kein Log-System. Die Idee dahinter ist, dass Du weitere Dependencies immer dann hinzufügen kannst, sobald Du diese benötigst. Warum solltest Du auf eine Template-Engine angewiesen sein, wenn Du eine HTTP-API oder ein CLI-Tool entwickelst?

Wie können wir weitere Dependencies hinzufügen? Mit Composer. Neben "normalen" Composer-Paketen werden wir mit zwei "besonderen" Paketen arbeiten:

* *Symfony-Komponenten*: Pakete, die Kernfunktionen und Abstraktionen auf niedriger Ebene implementieren, die die meisten Anwendungen benötigen (Routing, Konsole, HTTP-Client, Mailer, Cache, ...);

* *Symfony Bundles*: Pakete, die High-Level-Funktionen hinzufügen oder Integrationen mit Bibliotheken von Drittanbietern anbieten (Bundles werden hauptsächlich von der Community beigesteuert).

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Zuerst fügen wir den Symfony Profiler hinzu. Er ist ein gute Hilfe, wenn es darum geht, die Ursache eines Problems zu finden:

.. code-block:: bash

    $ symfony composer req profiler --dev

``profiler`` ist ein Alias für das ``symfony/profiler-pack``-Paket.

*Aliase* sind kein Feature von Composer, sondern ein Konzept von Symfony, das Dir das Leben leichter macht. Aliase sind Abkürzungen für beliebte Composer-Pakete. Möchtest Du ein ORM für Deine Anwendung? Nutze ``orm``. Möchtest Du eine API entwickeln? Nutze ``api``. Diese Aliase werden automatisch in ein oder mehrere reguläre Composer-Pakete aufgelöst. Bei den ausgewählten Paketen handelt es sich um persönliche Entscheidungen des Symfony-Core-Teams.

Ein weiteres nützliches Feature ist, dass Du den ``symfony``-vendor jederzeit weglassen kannst. Nutze ``cache`` statt ``symfony/cache``.

.. tip::

    Erinnerst Du dich, dass wir zuvor ein Composer-Plugin namens ``symfony/flex`` erwähnt haben? Aliase sind eines seiner Features.

Symfony-Environments verstehen
------------------------------

.. index::
    single: Symfony Environments

Hast Du das ``--dev``-Flag beim ``composer req``-Befehl bemerkt? Da der Symfony Profiler nur während der Entwicklung nützlich ist, wollen wir vermeiden, dass er in der Produktivumgebung installiert wird.

Symfony unterstützt den Umgang mit *Environments* (Umgebungen). Standardmäßig hat Symfony drei eingebaute Environments (``dev``, ``prod`` und ``test``) – Du kannst aber so viele hinzufügen, wie Du willst. Alle Environments teilen sich den gleichen Code, repräsentieren aber unterschiedliche *Konfigurationen*.

Beispielsweise sind alle Debugging-Tools in der ``dev``-Environment aktiviert. In der ``prod``-Environment ist die Anwendung auf Performance optimiert.

Der Wechsel von einer Environment zur anderen kann durch Ändern der Environment-Variable ``APP_ENV`` erfolgen.

Bei der Bereitstellung in der SymfonyCloud wurde die Environment (gespeichert in ``APP_ENV``) automatisch auf ``prod`` gesetzt.

Environment-Konfigurationen verwalten
-------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` kann durch die Verwendung von "echten" Environment-Variablen in Deinem Terminal festgelegt werden:

.. code-block:: bash
    :class: ignore

    $ export APP_ENV=dev

Die Verwendung von realen Environment-Variablen ist der bevorzugte Weg, um Werte wie ``APP_ENV`` auf Produktivsystemen zu setzen. Aber auf Entwicklungsmaschinen kann es mühsam sein, viele Environment-Variablen zu definieren. Definiere sie stattdessen in einer ``.env``-Datei.

Bei der Erstellung des Projekts wurde automatisch eine sinnvolle ``.env``-Datei für Dich generiert:

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    Jedes Paket kann dank seines von Symfony Flex verwendeten Recipes weitere Environment-Variablen zu dieser Datei hinzufügen.

Die ``.env``-Datei wird in das Repository commitet und beschreibt die *Standardwerte* für die Produktivumgebung. Du kannst diese Werte überschreiben, indem Du eine ``.env.local``-Datei erstellst. Diese Datei sollte nicht committet werden, weshalb sie in der ``.gitignore``-Datei bereits ignoriert wird.

Speichere niemals geheime oder sensible Werte in diesen Dateien. Wie solche Werte (Secrets) verwaltet werden, sehen wir in einem anderen Schritt.

Logging aller Dinge
-------------------

.. index::
    single: Logger

Bei neuen Projekten sind die Logging- und Debug-Möglichkeiten nach der Installation eingeschränkt. Lasst uns weitere Tools hinzufügen, die uns helfen, Probleme in der Entwicklung, aber auch im Produktivbetrieb zu untersuchen:

.. code-block:: bash

    $ symfony composer req logger

.. index::
    single: Components;Debug
    single: Debug

Wir sollten Debugging-Tools nur für die Entwicklung installieren:

.. code-block:: bash

    $ symfony composer req debug --dev

Symfony-Debugging-Tools entdecken
---------------------------------

Wenn Du die Homepage aktualisierst, solltest Du nun eine Symbolleiste am unteren Bildschirmrand sehen:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Das erste, was Du vielleicht bemerkst, ist das **404** in rot. Beachte, dass diese Seite ein Platzhalter ist, da wir noch keine Homepage definiert haben. Selbst wenn die Standardseite, die Dich begrüßt, schön ist, ist es immer noch eine Fehlerseite. Der richtige HTTP-Statuscode ist also 404 und nicht 200. Dank der Web-Debug-Toolbar hast Du diese Information sofort zur Hand.

Wenn du auf das kleine Ausrufezeichen klickst, erhältst du die "echte" Fehlermeldung als Teil der Logs im Symfony Profiler. Wenn Du den Stack Trace sehen möchtest, klicke auf den Link "Exception" im linken Menü.

Wann immer es ein Problem mit Deinem Code gibt, siehst Du eine Fehlerseite wie die Folgende, die Dir alles zeigt, was Du brauchst, um das Problem zu verstehen und herauszufinden woher es kommt:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Nimm Dir etwas Zeit, um die Informationen im Symfony Profiler zu erkunden.

.. index::
    single: Symfony CLI;server:log

Logs sind auch beim Debuggen von Sessions sehr nützlich. Symfony hat einen komfortablen Befehl, um alle Logs zu verfolgen (vom Webserver, PHP und Deiner Anwendung):

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Lass uns ein kleines Experiment machen. Öffne ``public/index.php`` und baue einen Fehler in den PHP-Code ein (füge z. B. foobar in der Mitte des Codes hinzu). Aktualisiere die Seite im Browser und beobachte den Log-Stream:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

Die Ausgabe ist schön gefärbt, um Deine Aufmerksamkeit auf Fehler zu lenken.

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Ein weiterer großartiger Debug-Helfer ist die Symfony-Funktion ``dump()``. Sie ist immer verfügbar und ermöglicht Dir, komplexe Variablen in einem schönen und interaktiven Format auszugeben.

Ändere vorübergehend ``public/index.php`` um das Request-Objekt auszugeben:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -18,5 +18,8 @@ if ($_SERVER['APP_DEBUG']) {
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);
    +
    +dump($request);
    +
     $response->send();
     $kernel->terminate($request, $response);

Beachte beim Aktualisieren der Seite das neue Fadenkreuz-Symbol in der Symbolleiste, mit dem Du die Debug-Ausgabe ansehen kannst. Klicke darauf, um die Vollansicht zu öffnen, auf der die Navigation einfacher wird:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Setze die Änderungen zurück, bevor Du die anderen Änderungen in diesem Schritt commitest:

.. code-block:: bash

    $ git checkout public/index.php

Konfiguration Deiner IDE
------------------------

Wenn in der Development-Environment (Entwicklungsumgebung) eine Exception ausgelöst wird, zeigt Symfony eine Seite mit der Fehlermeldung und deren Verlauf an. Wenn einen Dateipfad angezeigt wird, wird ein Link hinzugefügt, der die Datei in der richtigen Zeile in Deiner bevorzugten IDE öffnet. Um von dieser Funktion zu profitieren, musst Du Deine IDE konfigurieren. Symfony unterstützt viele IDEs direkt; ich verwende Visual Studio Code für dieses Projekt:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -6,3 +6,4 @@ max_execution_time=30
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

Dateien sind nicht nur bei Exceptions verlinkt - so wird beispielsweise auch der Controller in der Web-Debug-Toolbar nach der Konfiguration der IDE anklickbar.

Debuggen im Produktivsystem
---------------------------

.. index::
    single: SymfonyCloud;Remote Logs
    single: SymfonyCloud;SSH
    single: Symfony CLI;logs
    single: Symfony CLI;ssh

Das Debuggen von Produktivsystemen ist immer schwieriger. Du hast zum Beispiel keinen Zugriff auf den Symfony Profiler; Logs sind weniger ausführlich. Aber es ist möglich die Logs einzusehen:

.. code-block:: bash
    :class: ignore

    $ symfony logs

Du kannst Dich sogar über SSH zum Web-Container verbinden:

.. code-block:: bash
    :class: ignore

    $ symfony ssh

Keine Sorge, du kannst nicht einfach etwas kaputt machen. Der größte Teil des Dateisystems ist schreibgeschützt. Du wirst nicht in der Lage sein, einen Hotfix auf dem Produktivsystem durchzuführen. Aber du wirst später im Buch einen viel besseren Weg kennen lernen.

.. sidebar:: Weiterführendes

    * `SymfonyCasts-Tutorial zu Environments und Konfigurationsdateien <https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files>`_;

    * `SymfonyCasts-Tutorial zu Environment-Variablen <https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables>`_;

    * `SymfonyCasts-Tutorial zur Web-Debug-Toolbar und zum Profiler <https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler>`_;

    * `Verwaltung mehrerer .env-Dateien <https://symfony.com/doc/current/configuration.html#managing-multiple-env-files>`_ in Symfony-Anwendungen.
