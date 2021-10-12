Symfony Internals entdecken
===========================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

Wir verwenden Symfony schon seit geraumer Zeit, um eine leistungsstarke Anwendung zu entwickeln, aber der größte Teil des von der Anwendung ausgeführten Codes stammt von Symfony. Ein paar hundert Zeilen Code im Vergleich zu tausenden Zeilen Code.

Ich mag es zu verstehen, wie die Dinge hinter den Kulissen funktionieren. Und ich war schon immer fasziniert von Tools, die mir helfen zu verstehen, wie die Dinge funktionieren. Das erste Mal, als ich einen Schritt-für-Schritt Debugger benutzt habe oder das erste Mal ``ptrace`` entdeckte sind magische Erinnerungen.

Möchtest Du besser verstehen, wie Symfony funktioniert? Es ist an der Zeit, herauszufinden, wie Symfony Deine Anwendung zum Laufen bringt. Anstatt zu beschreiben, wie Symfony einen HTTP-Request aus theoretischer Sicht behandelt, was ziemlich langweilig wäre, werden wir Blackfire verwenden, um einige visuelle Darstellungen zu erhalten und um einige fortgeschrittenere Themen zu erkunden.

Symfony Internals mit Blackfire verstehen
-----------------------------------------

Du weißt bereits, dass alle HTTP-Requests von einem einzigen Einstiegspunkt verarbeitet werden: der ``public/index.php``-Datei. Aber was passiert als nächstes? Wie werden Controller aufgerufen?

Lass uns die englische Homepage in Production mit Blackfire über die Blackfire-Browsererweiterung analysieren:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

Oder direkt über die Kommandozeile:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Gehe zur "Timeline"-Ansicht des Profils. Du solltest etwas sehen, das dem Folgenden ähnlich sieht:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

Bewege den Mauszeiger über die farbigen Balken, um weitere Informationen zu jedem Anruf zu erhalten; Du wirst viel darüber lernen, wie Symfony funktioniert:

* Der Haupteinstiegspunkt ist ``public/index.php``;

* Die ``Kernel::handle()`` Methode behandelt die Anfrage;

* Sie ruft den ``HttpKernel`` auf, der einige Events wirft;

* Das erste Event ist ``RequestEvent``;

* Die ``ControllerResolver::getController()``-Methode wird aufgerufen, um zu bestimmen, welcher Controller für die eingehende URL aufgerufen werden soll;

* Die ``ControllerResolver::getArguments()``-Methode wird aufgerufen, um zu bestimmen, welche Argumente an den Controller übergeben werden sollen (der Parameter-Konverter wird aufgerufen);

* Die ``ConferenceController::index()``-Methode wird aufgerufen und der Großteil unseres Codes wird durch diesen Aufruf ausgeführt;

* Die ``ConferenceRepository::findAll()``-Methode holt alle Konferenzen aus der Datenbank (beachte die Verbindung zur Datenbank über ``PDO::__construct()``);

* Die ``Twig\Environment::render()``-Methode rendert das Template;

* Das ``ResponseEvent`` und das ``FinishRequestEvent`` werden ausgelöst, aber es sieht so aus, als ob keine Listener registriert sind, da sie sehr schnell verarbeitet sind.

Die Timeline ist eine gute Möglichkeit zu verstehen, wie der Code funktioniert; was sehr nützlich ist, wenn Du ein Projekt von jemand anderem entwickeln lässt.

Analysiere nun die gleiche Seite von der lokalen Maschine in der Development-Environment:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Öffne das Profil. Da die Anfrage sehr schnell war und die Timeline ziemlich leer wäre, solltest Du zur Call-Graph-Ansicht weitergeleitet werden:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Verstehst Du, was hier vor sich geht? Der HTTP-Cache ist aktiviert und deshalb analysieren wir die Symfony HTTP-Cache-Schicht. Da sich die Seite im Cache befindet, erhält ``HttpCache\Store::restoreResponse()`` die HTTP-Response aus seinem Cache und der Controller wird nie aufgerufen.

Deaktiviere die Cache-Ebene in ``public/index.php`` wie im vorherigen Schritt und versuche es erneut. Du siehst sofort, dass das Profil ganz anders aussieht:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Die Hauptunterschiede sind die folgenden:

* Das ``TerminateEvent``, welches in Production nicht sichtbar war, nimmt einen großen Teil der Ausführungszeit in Anspruch; bei genauerem Hinsehen siehst Du, dass dies das Event ist, das für die Speicherung der während der Anfrage gesammelten Symfony-Profilerdaten verantwortlich ist;

* Beachte unter dem ``ConferenceController::index()``-Aufruf die ``SubRequestHandler::handle()``-Methode, die das ESI rendert (deshalb haben wir zwei Aufrufe zu ``Profiler::saveProfile()``, einen für den Haupt-Request und einen für das ESI).

Erkunde die Timeline, um mehr zu erfahren; wechsle in die Call Graph-Ansicht, um eine andere Darstellung der gleichen Daten zu erhalten.

Wie wir gerade erfahren haben, ist der in Development und Production ausgeführte Code sehr verschieden. Die Development-Environment ist langsamer, da der Symfony-Profiler versucht, viele Daten zu sammeln, um das Debugging von Problemen zu erleichtern. Deshalb solltest Du für die Analyse immer die Production-Environment nutzen, auch lokal.

Einige interessante Experimente: Analysiere eine Fehlerseite, analysiere die ``/``-Seite (welche ein Redirect ist), oder eine API-Ressource. Jedes Profil wird dich etwas mehr darüber lehren, wie Symfony funktioniert, welche Klassen/Methoden aufgerufen werden, was teuer und was billig in der Ausführung ist.

Das Blackfire Debug Addon verwenden
-----------------------------------

.. index::
    single: Blackfire;Debug Addon

Um große Nutzlasten und große Graphen zu vermeiden entfernt Blackfire standardmäßig alle Methodenaufrufe, die nicht signifikant genug sind. Wenn Du Blackfire als Debugging-Tool verwendest, ist es besser, alle Aufrufe zu behalten. Dies wird durch das Debug-Addon ermöglicht.

Verwende von der Befehlszeile aus das ``--debug``-Flag:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

In Production siehst Du z.B. das Laden einer Datei namens ``.env.local.php``:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

Wo kommt sie her? SymfonyCloud führt einige Optimierungen bei der Bereitstellung einer Symfony-Anwendung durch, wie z. B. die Optimierung des Composer-Autoloaders (``--optimize-autoloader --apcu-autoloader --classmap-authoritative``). Es optimiert auch die in der ``.env``-Datei definierten Environment-Variablen (um zu vermeiden, dass die Datei für jede Anforderung geparst wird), indem es die ``.env.local.php``-Datei erzeugt:

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire ist ein sehr mächtiges Tool, das zu verstehen hilft, wie Code von PHP ausgeführt wird. Die Verbesserung der Performance ist nur eine Möglichkeit, einen Profiler zu nutzen.

Einen Schritt-für-Schritt Debugger (Step Debugger) mit Xdebug nutzen
--------------------------------------------------------------------

.. index::
    single: Xdebug
    single: Debugger

Blackfire Timelines und Call-Graph-Ansichten erlauben Entwickler*innen zu visualisieren welche Dateien/Funktionen/Methoden von der PHP-Engine ausgeführt werden, um besser die Code-Basis des Projektes zu verstehen.

Ein anderer Weg um der Code-Ausführung zu folgen ist ein **step debugger** wie `Xdebug <https://xdebug.org>`_. Ein solcher Debugger erlaubt es Entwickler*innen interaktiv und Schritt für Schritt den Code eines PHP-Projektes zu durchlaufen, um den Control Flow (Ablauf) zu debuggen und Datenstrukturen zu untersuchen. Er ist sehr hilfreich um unerwartetes Verhalten zu debuggen und ersetzt die übliche "var_dump()/exit()"-Debugging-Technik.

Installiere zuerst die ``xdebug``-PHP-Erweiterung. Kontrolliere mit diesem Befehl ob sie installiert ist:

.. code-block:: bash

    $ symfony php -v

Du solltest Xdebug in der Ausgabe sehen:

.. code-block:: text
    :emphasize-lines: 5
    :class: ignore

    PHP 8.0.1 (cli) (built: Jan 13 2021 08:22:35) ( NTS )
    Copyright (c) The PHP Group
    Zend Engine v4.0.1, Copyright (c) Zend Technologies
        with Zend OPcache v8.0.1, Copyright (c), by Zend Technologies
        with Xdebug v3.0.2, Copyright (c) 2002-2021, by Derick Rethans
        with blackfire v1.49.0~linux-x64-non_zts80, https://blackfire.io, by Blackfire

Du kannst auch im Browser kontrollieren, ob Xdebug für PHP-fPM aktiviert ist, in dem Du auf den "View phpinfo()"-Link klickst, wenn Du über das Symfony Logo in der Web-Debug-Toolbar mit der Maus drüberfährst:

.. figure:: screenshots/phpinfo.png
    :alt: /
    :align: center
    :figclass: with-browser

Ok, aktiviere jetzt den ``debug``-Mode von Xdebug:

.. code-block:: ini
    :caption: php.ini
    :class: ignore

    [xdebug]
    xdebug.mode=debug
    xdebug.start_with_request=yes

Standardmäßig schickt Xdebug Daten zum Port 9003 des lokalen Host.

Xdebug kann man auf viele Arten auslösen, aber am einfachsten ist Xdebug von Deiner IDE zu bedienen. In diesem Kapitel werden wir Visual Studio Code nutzen um zu zeigen wie es funktioniert. Installiere die `PHP Debug <https://marketplace.visualstudio.com/items?itemName=felixfbecker.php-debug>`_-Erweiterung durch das Starten der "Quick Open"-Funktion (``Ctrl+P``), füge den folgenden Befehl ein und drück Enter:

.. code-block:: text
    :class: ignore

    ext install felixfbecker.php-debug

Erstelle die folgende Konfigurations-Datei:

.. code-block:: json
    :caption: .vscode/launch.json
    :emphasize-lines: 8,16
    :class: ignore

    {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Listen for XDebug",
                "type": "php",
                "request": "launch",
                "port": 9003
            },
            {
                "name": "Launch currently open script",
                "type": "php",
                "request": "launch",
                "program": "${file}",
                "cwd": "${fileDirname}",
                "port": 9003
            }
        ]
    }

Gehe innerhalb Visual Studio Code und in Deinem Projekt-Verzeichnis zu dem Debugger und klick auf den grünen play-Button der mit "Listen for Xdebug" beschriftet ist:

.. figure:: images/vs-xdebug-run.png
    :align: center

Wenn Du in Deinem Browser die Seite aktualisierst, sollte die IDE automatisch in den Vordergrund kommen. Dies bedeutet, dass die Debugging-Session bereit ist. Standardmäßig ist alles ein Breakpoint (Haltepunkt), weshalb die Ausführung beim ersten Befehl stoppt. Es liegt dann an Dir, die aktuellen Variablen zu prüfen, über den Code drüber zu gehen, in den Code reinzugehen, ...

Während des Debugging kannst Du den "Everything"-Breakpoint deaktivieren und selbst Breakpoints in Deinem Code definieren.

Wenn Du noch nie mit Schritt-für-Schritt Debuggern gearbeitet hast, lese die `ausgezeichneten Tutorials für Visual Studio Code <https://code.visualstudio.com/Docs/editor/debugging>`_, welche alles visuell erklären.

.. sidebar:: Weiterführendes

    * `Die Xdebug Step Debugging Dokumentation <https://xdebug.org/docs/step_debug>`_;

    * `Debugging mit Visual Studio Code <https://code.visualstudio.com/Docs/editor/debugging>`_.
