Vom Nichts zum Produktivbetrieb
===============================

Ich mag schnelle Ergebnisse. Deshalb möchte ich, dass unser kleines Projekt so schnell wie möglich live geschaltet wird. Und zwar genau jetzt. Im Produktivbetrieb. Da wir noch nichts entwickelt haben, werden wir zunächst eine schöne und einfache "Under Construction"-Seite einrichten. Du wirst es lieben!

Nimm Dir Zeit dafür das ideale, altmodische und animierte "Under Construction" GIF im Internet zu finden. Hier ist `was, <http://clipartmag.com/images/website-under-construction-image-6.gif>`_ ich benutzen werde:

.. image:: images/under-construction.gif
    :align: center

Ich habe Dir ja gesagt, dass es eine Menge Spaß machen wird.

Das Projekt initialisieren
--------------------------

Erstelle ein neues Symfony-Projekt mit dem ``symfony`` CLI-Tool, das wir zuvor gemeinsam installiert haben:

.. code-block:: bash

    $ symfony new guestbook --version=5.0
    $ cd guestbook

Dieser Befehl ist ein dünner Wrapper von ``Composer`` der die Erstellung von Symfony-Projekten erleichtert. Er verwendet ein `Projektskelett <https://github.com/symfony/skeleton>`_, das nur die allernötigsten Abhängigkeiten enthält; die Symfony-Komponenten, die für fast jedes Projekt benötigt werden: ein Konsolenwerkzeug und die HTTP-Abstraktion, die für die Erstellung von Webanwendungen erforderlich ist.

Wenn Du Dir das GitHub-Repository vom Skeleton ansiehst, wirst Du feststellen, dass es fast leer ist. Nur eine ``composer.json`` Datei. Aber das ``guestbook`` Verzeichnis ist voller Dateien. Wie ist das überhaupt möglich? Die Antwort liegt im ``symfony/flex``-Paket. Symfony Flex ist ein Composer-Plugin, das sich in den Installationsprozess einfügt. Wenn es ein Paket erkennt, für das es ein *Recipe* (Rezept) gibt, wird dieses ausgeführt.

Der wichtigste Einstiegspunkt eines Symfony-Recipes ist eine Manifestdatei, welche die Vorgänge beschreibt, die durchgeführt werden müssen, um das Paket automatisch in einer Symfony-Anwendung zu registrieren. Du musst nie eine README-Datei lesen, um ein Paket mit Symfony zu installieren. Automatisierung ist ein wesentliches Merkmal von Symfony.

Da Git auf unserer Maschine installiert ist, hat ``symfony new`` auch ein Git-Repository für uns erstellt und den allerersten Commit hinzugefügt.

Wirf einen Blick auf die Verzeichnisstruktur:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

Das ``bin/`` Verzeichnis enthält den wichtigsten CLI-Einstiegspunkt: ``console``. Du wirst ihn ständig verwenden.

Das ``config/`` Verzeichnis besteht aus einer Reihe von sinnvollen Standard-Konfigurationsdateien. Eine Datei pro Paket. Du wirst sie kaum ändern, den Standardeinstellungen zu vertrauen ist fast immer eine gute Idee.

Das ``public/``-Verzeichnis ist das Web-Root-Verzeichnis und das ``index.php``-Skript ist der Einstiegspunkt für alle dynamischen HTTP-Ressourcen.

Das ``src/`` Verzeichnis enthält den gesamten Code, den Du schreiben wirst; dort wirst Du die meiste Zeit verbringen. Standardmäßig verwenden alle Klassen in diesem Verzeichnis den ``App`` PHP-Namespace. Es ist Dein Zuhause. Dein Code. Deine Domänenlogik. Symfony hat dort sehr wenig zu sagen.

Das ``var/`` Verzeichnis enthält Caches, Logs und Dateien, die zur Laufzeit von der Anwendung generiert werden. Dieses kannst Du getrost in Ruhe lassen. Es ist das einzige Verzeichnis, das im Produktivbetrieb beschreibbar sein muss.

Das ``vendor/`` Verzeichnis enthält alle von Composer installierten Pakete, einschließlich Symfony selbst. Es ist unsere Geheimwaffe, um produktiver zu sein. Lass uns das Rad nicht neu erfinden. Du wirst dich auf bestehende Bibliotheken verlassen, die die harte Arbeit für dich erledigen. Dieses Verzeichnis wird vom Composer verwaltet, also niemals anfassen.

Das ist alles, was Du im Moment wissen musst.

Öffentliche Ressourcen erstellen
---------------------------------

Alles, was unter ``public/`` liegt, ist über einen Browser zugänglich. Wenn Du beispielsweise Deine animierte GIF-Datei (Name ``under-construction.gif``) in ein neues ``public/images/`` Verzeichnis verschiebst, ist sie unter einer URL wie ``https://localhost/images/under-construction.gif`` erreichbar.

Lade mein GIF-Bild hier herunter:

.. code-block:: bash

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Einen lokalen Web-Server starten
--------------------------------

.. index::
    single: Symfony CLI;server:start

Im Lieferumfang der ``symfony`` CLI ist ein Webserver enthalten, der für die Entwicklungsarbeit optimiert ist. Es wird Dich nicht überraschen, wenn ich Dir sage, dass er für Symfony reibungslos funktioniert. Verwende ihn jedoch niemals im Produktivbetrieb.

Starte, vom Projektverzeichnis aus, den Webserver im Hintergrund (``-d`` Flag):

.. code-block:: bash

    $ symfony server:start -d

Der Server startete auf dem ersten verfügbaren Port, beginnend mit 8000. Als Abkürzung öffne die Webseite über die CLI in einem Browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Dein bevorzugter Browser sollte in den Vordergrund kommen und eine neue Registerkarte öffnen, die etwas Ähnliches wie das Folgende anzeigt:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    Um Probleme zu beheben, führe ``symfony server:log`` aus; es verfolgt die Protokolle vom Webserver, PHP und Deiner Anwendung.

Gehe auf ``/images/under-construction.gif``. Sieht es so aus?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

Zufrieden? Lasst uns unsere Arbeit committen:

.. code-block:: bash
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Ein Favicon hinzufügen
-----------------------

Um zu vermeiden, dass 404 HTTP-Fehler die Logs "zuspammen" wegen eines fehlenden Favicon, das von Browsern angefordert wird, fügen wir jetzt eins hinzu:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/favicon.ico', 'public/favicon.ico');"
    $ git add public/
    $ git commit -m'Add a favicon'

Den Produktivbetrieb vorbereiten
--------------------------------

.. index::
    single: SymfonyCloud;Initialization

Wie sieht es mit dem Deployment in die Produktivumgebung aus? Ich weiß, wir haben noch nicht einmal eine eigene HTML-Seite, um unsere Benutzer*innen zu begrüßen. Aber das kleine "Under Construction"-Bild auf einem Produktivserver sehen zu können, wäre ein großer Schritt nach vorne. Und Du kennst das Motto: *deploy early and often*.

Du kannst diese Anwendung bei jedem Provider hosten, der PHP unterstützt... also bei fast allen Hosting-Providern. Überprüfe jedoch ein paar Dinge: Wir wollen die neueste PHP-Version und die Möglichkeit, Dienste wie eine Datenbank, eine Queue und einiges mehr zu hosten.

Ich habe meine Wahl getroffen, es wird die `SymfonyCloud <https://symfony.com/cloud>`_ sein. Sie bietet alles, was wir brauchen, und hilft die Entwicklung von Symfony zu finanzieren.

.. index::
    single: Symfony CLI;project:init

Die ``symfony`` CLI verfügt über eine integrierte Unterstützung für die SymfonyCloud. Lass uns ein SymfonyCloud-Projekt initialisieren:

.. code-block:: bash

    $ symfony project:init

Dieser Befehl erstellt einige Dateien, die von der SymfonyCloud benötigt werden, nämlich ``.symfony/services.yaml``, ``.symfony/routes.yaml`` und ``.symfony.cloud.yaml``.

Füge sie zu Git hinzu und committe:

.. code-block:: bash

    $ git add .
    $ git commit -m"Add SymfonyCloud configuration"

.. note::

    Die Verwendung des generischen und gefährlichen ``git add .``  funktioniert einwandfrei, da eine ``.gitignore`` Datei generiert wurde, die automatisch alle Dateien ausschließt, die wir nicht übertragen wollen.

Der Weg zum Produktivsystem
---------------------------

.. index::
    single: Symfony CLI;project:create
    single: Symfony CLI;deploy

Zeit zu deployen?

Erstelle ein neues SymfonyCloud-Projekt:

.. code-block:: bash

    $ symfony project:create --title="Guestbook" --plan=development

Dieser Befehl macht viel:

* Wenn Du diesen Befehl zum ersten Mal ausführst, dann authentifiziere Dich mit Deinen SymfonyConnect-Zugangsdaten, falls noch nicht geschehen.

* Er stellt ein neues Projekt auf der SymfonyCloud bereit (Du erhältst 7 Tage *kostenlos* für jedes neue Entwicklungsprojekt).

Dann deploye:

.. code-block:: bash

    $ symfony deploy

Der Code wird durch das Pushen des Git-Repository bereitgestellt. Nach der Ausführung des Befehls hat das Projekt einen bestimmten Domainnamen, mit dem Du darauf zugreifen kannst.

.. index::
    single: Symfony CLI;open:remote

Überprüfe, ob alles geklappt hat:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Du solltest eine 404er Fehlerseite bekommen, aber das browsen zu ``/images/under-construction.gif`` sollte unsere Arbeit enthüllen.

Beachte, dass Du nicht die schöne Standard-Symfony-Seite auf der SymfonyCloud erhältst. Warum? Du wirst bald feststellen, dass Symfony mehrere Environments (Umgebungen) unterstützt und SymfonyCloud den Code automatisch in der Production-Environment (Produktivumgebung) bereitgestellt hat.

.. index::
    single: Symfony CLI;project:delete

.. tip::

    Wenn Du das Projekt in der SymfonyCloud löschen möchtest, verwende den ``project:delete``-Befehl.

.. sidebar:: Weiterführendes

    * Der `Symfony Recipes Server <https://flex.symfony.com/>`_, auf dem Du alle verfügbaren Recipes für Deine Symfony-Anwendungen findest;

    * Die Repositories für die `offiziellen Symfony-Recipes <https://github.com/symfony/recipes>`_ und für die von `der Community beigesteuerten Recipes <https://github.com/symfony/recipes-contrib>`_, wo Du deine eigenen Recipes einreichen kannst;

    * Der `lokale Symfony Webserver <https://symfony.com/doc/current/setup/symfony_server.html>`_;

    * Die `SymfonyCloud-Dokumentation <https://symfony.com/doc/cloud>`_.
