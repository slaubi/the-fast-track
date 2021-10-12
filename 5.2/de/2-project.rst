Vorstellung des Projekts
========================

Wir müssen ein Projekt finden, an dem wir arbeiten können. Es ist eine ziemliche Herausforderung, da wir ein Projekt finden müssen, das groß genug ist, um Symfony vollständig abzudecken, aber gleichzeitig auch klein genug sein sollte; ich möchte nicht, dass du dich langweilst während du ähnliche Funktionen mehrfach implementierst.

Enthüllung des Projekts
-----------------------

Da das Buch während der SymfonyCon Amsterdam veröffentlicht werden muss, wäre es schön, wenn das Projekt irgendwie mit Symfony und Konferenzen zu tun hat. Wie wäre es mit einem `Gästebuch <https://en.wikipedia.org/wiki/Guestbook>`_? Ein `Livre d'or <https://fr.wikipedia.org/wiki/Livre_d%27or>`_, wie wir auf Französisch sagen. Ich mag das altmodische und veraltete Gefühl, ein Gästebuch im Jahr 2019 zu entwickeln!

Wir haben es. Bei dem Projekt geht es darum, Feedback zu Konferenzen zu erhalten: eine Liste der Konferenzen auf der Homepage, eine Seite für jede Konferenz, voller schöner Kommentare. Ein Kommentar besteht aus einem kleinen Text und einem optionalen Foto, das während der Konferenz aufgenommen wurde. Ich schätze, ich habe gerade alle Spezifikationen aufgeschrieben, die wir brauchen, um loszulegen.

Das *Projekt* wird mehrere *Anwendungen* umfassen. Eine traditionelle Webanwendung mit einem HTML-Frontend, einer API und einem SPA für Mobiltelefone. Wie klingt das?

Lernen durch Handeln
--------------------

Lernen durch Handeln. Punkt. Ein Buch über Symfony zu lesen ist schön. Noch besser ist es, eine Anwendung auf Deinem PC zu programmieren und gleichzeitig ein Buch über Symfony zu lesen. Das besondere an diesem Buch ist, dass alles getan wurde, damit Du es mitverfolgen, nachprogrammieren und sicher sein kannst, die gleichen Ergebnisse zu erzielen, die ich lokal auf meiner Maschine hatte, als ich es anfangs entwickelte.

Das Buch enthält allen Code, den Du schreiben, und alle Befehle, die Du ausführen musst, um das Endergebnis zu erhalten. Es fehlt kein Code. Alle Befehle stehen in diesem Buch. Dies ist möglich, da moderne Symfony-Anwendungen nur sehr wenig Boilerplate-Code haben. Der größte Teil des Codes, den wir zusammen schreiben werden betrifft die *Geschäftslogik* des Projekts. Alles andere ist weitgehend automatisiert oder wird für uns automatisch generiert.

Blick auf das endgültige Infrastrukturdiagramm
----------------------------------------------

Auch wenn die Projektidee einfach erscheint, werden wir kein "Hello World"-ähnliches Projekt bauen. Außerdem werden wir nicht nur PHP und eine Datenbank verwenden.

Ziel ist es, ein Projekt mit einigen der Komplexitäten zu erstellen, die Du in auch der Praxis finden kannst. Du willst einen Beweis? Dann wirf einen Blick auf die endgültige Infrastruktur des Projekts:

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

Einer der großen Vorteile der Verwendung eines Frameworks ist die geringe Menge an Code, die für die Entwicklung eines solchen Projekts benötigt wird:

* 20 PHP-Klassen unter ``src/`` für die Website;

* 550 PHP Logical Lines of Code (LLOC) gemäß `PHPLOC <https://github.com/sebastianbergmann/phploc>`_;

* 40 Zeilen Konfigurationsanpassungen in 3 Dateien (über Annotations und YAML), hauptsächlich für das Design des Backends;

* 20 Zeilen Konfiguration der Entwicklungsinfrastruktur (Docker);

* 100 Zeilen Konfiguration der Produktivinfrastruktur (SymfonyCloud);

* 5 explizite Environment-Variablen (Umgebungsvariablen).

Bereit für die Herausforderung?

Holen des Projekt-Quellcodes
----------------------------

Um auf altmodische Art und Weise fortzufahren, hätte ich einfach eine CD mit dem Quellcode erstellen können, oder? Wie wäre es aber stattdessen mit einem begleitenden Git-Repository?

.. index::
    single: Project;Git Repository
    single: Git;clone

Klone das `Gästebuch-Repository <https://github.com/the-fast-track/book-5.0-1>`_ irgendwo auf Deinem lokalen Rechner:

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.0-1 --book guestbook

Dieses Repository enthält den gesamten Code aus diesem Buch.

Beachte, dass wir anstelle von ``git clone`` den Befehl ``symfony new`` verwenden, weil der Befehl mehr macht als einfach nur das Repository zu klonen (gehostet auf Github unter der ``the-fast-track`` Organisation: ``https://github.com/the-fast-track/book-5.0-1``). Er startet auch den Webserver, die Container, migriert die Datenbank und lädt die Fixtures. Nach dem Ausführen des Befehls sollte die Website betriebs- und einsatzbereit sein.

Der Code ist zu 100% mit dem Code im Buch synchronisiert (verwende dafür die genaue Repository-URL, die oben aufgeführt ist). Der Versuch, Änderungen aus dem Buch manuell mit dem Quellcode im Repository zu synchronisieren, ist fast unmöglich. Ich habe es in der Vergangenheit versucht und dabei versagt. Es ist einfach unmöglich. Besonders für Bücher wie die, die ich schreibe: Solche, die Dir eine Geschichte über die Entwicklung einer Website erzählen. Da jedes Kapitel von den vorherigen abhängt, kann eine Änderung in allen folgenden Kapiteln Konsequenzen haben.

Die gute Nachricht ist, dass das Git-Repository für dieses Buch *automatisch* aus dem Buchinhalt *generiert* wird. Ja, Du hast richtig gelesen. Ich automatisiere gerne alles, deshalb gibt es ein Skript, dessen Aufgabe es ist, das Buch zu lesen und das Git-Repository zu erstellen. Dadurch entsteht ein schöner Nebeneffekt: Beim Aktualisieren des Buches schlägt das Skript fehl, wenn die Änderungen inkonsistent sind oder wenn ich es vergesse, irgendwelche Anweisungen zu aktualisieren. Das ist BDD, Book Driven Development!

Navigieren im Quellcode
-----------------------

Noch besser, das Repository enthält mehr als nur die endgültige Version des Codes auf dem ``master`` branch. Das Skript führt jede im Buch erklärte Aktion aus und committet am Ende jedes Abschnitts seine Arbeit. Es markiert auch jeden Schritt und Teilschritt, um das Durchsuchen des Codes zu erleichtern. Schön, nicht wahr?

.. index::
    single: Git;checkout

Wenn Du faul bist, kannst Du den Zustand des Codes am Ende eines Schrittes erhalten, indem Du den richtigen Tag auscheckst. Wenn Du beispielsweise den Code am Ende von Schritt 10 lesen und testen möchtest, führe Folgendes aus:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Wie beim Klonen des Repositorys verwenden wir nicht ``git checkout`` sondern ``symfony book:checkout``. Der Befehl stellt sicher, dass Du, egal in welchem Schritt Du Dich gerade befindest, am Ende eine funktionierende Website für den von Dir gewünschten Schritt erhältst. **Beachte, dass alle Daten, Code und Container durch diesen Vorgang entfernt werden.**

Du kannst auch jeden Teilschritt auschecken:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

Nochmals, ich empfehle Dir dringend, selbst mitzuprogrammieren. Aber wenn Du nicht weiterkommst, kannst Du das, was Du hast, immer mit dem Inhalt des Buches vergleichen.

.. index::
    single: Git;diff

Du bist Dir nicht sicher, ob Du in Teilschritt 10.2 alles richtig gemacht hast? Lass Dir den Unterschied anzeigen:

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Möchtest Du wissen, wann eine Datei erstellt oder geändert wurde?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

Du kannst auch Diffs, Tags und Commits direkt auf GitHub durchsuchen. Dies ist eine großartige Möglichkeit, Code zu kopieren und einzufügen, wenn Du ein Buch aus Papier liest!
