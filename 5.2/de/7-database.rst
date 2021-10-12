Eine Datenbank einrichten
=========================

.. index::
    single: Database

Auf der Website des Konferenzgästebuchs geht es darum, während der Konferenzen Feedback zu sammeln. Wir müssen die Kommentare von den Konferenzteilnehmer*innen in einem permanenten Speicher ablegen.

Ein Kommentar lässt sich am besten durch eine feste Datenstruktur beschreiben: Autor*in, E-Mail, der Text des Feedbacks und ein optionales Foto. Das ist die Art Daten, die am besten in einer traditionellen relationalen Datenbank gespeichert wird.

PostgreSQL ist das Datenbanksystem unserer Wahl.

PostgreSQL zu Docker Compose hinzufügen
----------------------------------------

.. index::
    single: Docker;PostgreSQL

Auf unserem lokalen Rechner haben wir uns entschieden, Docker zur Verwaltung von Services zu verwenden. Erstelle eine ``docker-compose.yaml``-Datei und füge PostgreSQL als Service hinzu:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

Dadurch wird ein PostgreSQL-Server installiert und einige Environment-Variablen konfiguriert, die den Datenbanknamen und die Credentials (Anmeldeinformationen) steuern. Die Werte spielen keine Rolle.

Wir stellen auch den PostgreSQL-Port (``5432``) des Containers dem lokalen Host zur Verfügung. Das wird uns helfen, von unserer Maschine aus auf die Datenbank zuzugreifen.

.. note::

    Die ``pdo_pgsql``-Erweiterung sollte installiert worden sein, als PHP in einem vorherigen Schritt eingerichtet wurde.

Docker Compose starten
----------------------

Starte Docker Compose im Hintergrund (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Warte ein wenig, bis die Datenbank hochgefahren ist und überprüfe, ob alles in Ordnung ist:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Überprüfe die Docker Compose logs wenn es keine aktiven Container gibt oder wenn die ``State``-Spalte nicht ``Up`` anzeigt:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Zugriff auf die lokale Datenbank
--------------------------------

Die Verwendung des ``psql``-Befehls im Terminal kann sich von Zeit zu Zeit als nützlich erweisen. Aber Du musst dir die Anmeldeinformationen und den Datenbanknamen merken. Weniger offensichtlich ist, dass Du auch den lokalen Port kennen musst, mit dem die Datenbank auf dem Host läuft. Docker wählt einen zufälligen Port, so dass Du an mehr als einem Projekt gleichzeitig mit PostgreSQL arbeiten kannst (der lokale Port ist Teil der Ausgabe von ``docker-compose ps``).

Wenn Du ``psql`` über die Symfony CLI  aufrufst, musst Du dir nichts merken.

Die Symfony CLI erkennt automatisch die für das Projekt ausgeführten Docker-Dienste und stellt die Environment-Variablen bereit, die ``psql`` für die Verbindung zur Datenbank benötigt.

.. index::
    single: Symfony CLI;run psql

Dank dieser Konventionen ist der Zugriff auf die Datenbank via ``symfony run`` viel einfacher:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    Wenn Du die ``psql`` Binärdatei nicht auf Deinem lokalen Host hast, kannst Du sie auch über ``docker-compose`` laufen lassen:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Datenbank-Daten exportieren und importieren
-------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Verwende ``pg_dump`` um did Datenbank-Daten zu exportieren:

.. code-block:: bash
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

Und importiere die Daten mit:

.. code-block:: bash
    :class: ignore

    $ symfony run psql < dump.sql

.. warning::

    Führe nie ``docker-compose down`` aus, wenn Du Deine Daten nicht verlieren möchtest. Oder mach erst ein Backup.

PostgreSQL zur SymfonyCloud hinzufügen
---------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Für die Produktiv-Infrastruktur auf SymfonyCloud sollte das Hinzufügen eines Dienstes wie PostgreSQL in der aktuell leeren ``.symfony/services.yaml``-Datei erfolgen:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Der ``db`` Dienst ist eine PostgreSQL-Datenbank (in der selben Version wie für Docker), die wir auf einem kleinen Container mit einer Kapazität von 1 GB bereitstellen wollen.

Wir müssen auch die DB mit dem Anwendungscontainer "verknüpfen", was in ``.symfony.cloud.yaml`` beschrieben ist:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Der ``db`` Dienst vom Typ ``postgresql`` wird  auf dem Anwendungscontainer mit ``database`` referenziert.

Der letzte Schritt besteht darin, die ``pdo_pgsql`` Erweiterung zur PHP-Laufzeit hinzuzufügen:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Das sind alle Unterschiede in der ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

Commite diese Änderungen und deploye sie dann erneut zur SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Zugriff auf die SymfonyCloud-Datenbank
--------------------------------------

PostgreSQL läuft nun sowohl lokal über Docker als auch in der Produktivumgebung auf SymfonyCloud.

Wie wir gerade gesehen haben, verbindet ``symfony run psql`` sich automatisch mit der von Docker gehosteten Datenbank – Dank der Environment-Variablen, die von ``symfony run`` bereitgestellt werden.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Wenn Du eine Verbindung zur PostgreSQL-Datenbank herstellen möchtest, die auf den Production-Containern gehostet wird, kannst Du einen SSH-Tunnel zwischen dem lokalen Computer und der SymfonyCloud-Infrastruktur öffnen:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

Standardmäßig werden SymfonyCloud-Dienste nicht als Environment-Variablen auf dem lokalen Rechner angezeigt. Dies muss explizit durch die Verwendung des ``--expose-env-vars``-Flags erfolgen. Warum? Die Verbindung zur Datenbank in der Produktivumgebung ist ein gefährlicher Vorgang. Du kannst mit *echten* Daten herumpfuschen. Indem Du das Flag nutzt, bestätigst Du, dass Du dies *wirklich* tun möchtest.

Verbinde dich nun via ``symfony run psql`` wie bisher mit der remote PostgreSQL-Datenbank:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Wenn du fertig bist, vergiss nicht, den Tunnel zu schließen:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Um einige SQL-Abfragen auf der Production-Datenbank auszuführen, kannst Du auch den ``symfony sql``-Befehl anstelle einer Shell verwenden.

Environment-Variablen bereitstellen
-----------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose und SymfonyCloud arbeiten dank Environment-Variablen nahtlos mit Symfony zusammen.

Überprüfe alle Environment-Variablen, die durch ``symfony`` bereitgestellt werden, indem Du ``symfony var:export`` ausführst:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Die ``PG*`` Environment-Variablen werden vom ``psql`` Dienstprogramm gelesen. Was ist mit den anderen?

Wenn ein Tunnel zu SymfonyCloud mit gesetztem ``--expose-env-vars``-Flag geöffnet ist, gibt der ``var:export``-Befehl die Environment-Variablen in der SymfonyCloud zurück:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

Deine Infrastruktur beschreiben
-------------------------------

Du hast es vielleicht noch nicht gemerkt, aber es ist sehr hilfreich die Infrastruktur mit beim Code zu speichern. Docker und SymfonyCloud nutzen Konfigurationsdateien um die Infrastruktur des Projektes zu beschreiben. Wenn eine neue Funktionalität einen zusätzlichen Service braucht, sind die Änderungen des Code und die Änderungen für die Infrastruktur im selben Patch.

.. sidebar:: Weiterführendes

    * `SymfonyCloud-Dienste <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud-Tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `PostgreSQL-Dokumentation <https://www.postgresql.org/docs/>`_;

    * `docker-compose Befehle <https://docs.docker.com/compose/reference/>`_.
