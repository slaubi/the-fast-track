Den Code branchen
=================

Es gibt viele Möglichkeiten, den Workflow von Code-Änderungen in einem Projekt zu organisieren – Das direkte Arbeiten am Git Master-Branch und der direkte Einsatz im Produktivbetrieb ohne Tests ist aber nicht der beste Weg.

Beim Testen geht es nicht nur um Unit- oder Funktionale Tests, sondern auch um die Überprüfung des Anwendungsverhaltens mit echten Daten. Wenn Du oder Deine `Stakeholder`_ die Anwendung genau so benutzen können, wie sie für Endbenutzer*innen bereitgestellt wird, entsteht ein großer Vorteil, der Dir ermöglicht die Anwendung mit Vertrauen deployen zu können. Es ist besonders effizient, wenn nicht-technische Personen neue Funktionen validieren können.

Wir werden der Einfachheit halber (und zur Vermeidung von Wiederholungen) die nächsten Schritte auf dem Git Master-Branch fortführen, aber mal sehen, wie das besser funktionieren könnte.

Einen Git-Workflow einführen
-----------------------------

Ein möglicher Workflow ist die Erstellung eines Branches pro neuem Feature oder Bugfix. Das ist einfach und effizient.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Branches erstellen
------------------

.. index::
    single: Git;branch
    single: Git;checkout

Der Workflow beginnt mit der Erstellung eines Git-Branches:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Dieser Befehl erstellt einen ``sessions-in-redis``-Branch, basierend auf dem ``master``-Branch. Es "spaltet" den Code und die Infrastrukturkonfiguration vom ``master``-Branch ab.

Sessions in Redis speichern
---------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Wie Du vielleicht schon am Namen des Branches erraten hast, wollen wir die Speicherung der Sessions vom Dateisystem auf Redis umstellen.

Die notwendigen Schritte, um dies zu verwirklichen, sind typisch:

#. Erstelle einen Git-Branch;

#. Aktualisiere bei Bedarf die Symfony-Konfiguration;

#. Schreibe und/oder aktualisiere bei Bedarf etwas Code;

#. Aktualisiere die PHP-Konfiguration (füge die Redis PHP-Erweiterung hinzu);

#. Aktualisiere die Infrastruktur auf Docker und SymfonyCloud (füge den Redis-Dienst hinzu);

#. Teste lokal;

#. Teste remote;

#. Führe den Branch mit dem Master-Branch zusammen;

#. Deploye in die Produktivumgebung;

#. Lösche den Branch.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Ich lasse Dich lokal testen, indem Du Dir die Website anschaust. Da es keine visuellen Veränderungen gibt und wir noch keine Sessions verwenden, sollte alles wie bisher funktionieren.

Einen Branch deployen
---------------------

.. index::
    single: SymfonyCloud;Environment

Bevor wir zum Produktivsystem deployen, sollten wir den Branch auf der gleichen Infrastruktur wie die Production-Environment testen. Wir sollten auch sicherstellen, dass für die Symfony ``prod``-Environment alles gut funktioniert (die lokale Website hat die Symfony ``dev``-Environment verwendet).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Lasst uns nun eine *SymfonyCloud-Environment* erstellen, die auf dem *Git-Branch* basiert:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Dieser Befehl erstellt eine neue Environment:

* Der Branch erbt den Code und die Infrastruktur vom aktuellen Git-Branch (``sessions-in-redis``);

* Die Daten stammen von der Master-Environment (auch bekannt als Production oder Produktivumgebung), und zwar durch eine Momentaufnahme aller Servicedaten, einschließlich Dateien (z. B. von Benutzer*innen hochgeladene Dateien) und Datenbanken;

* Ein neuer dedizierter Cluster wird erstellt, um den Code, die Daten und die Infrastruktur zu deployen.

Da das Deployment den gleichen Schritten folgt wie das Deployment in die Produktivumgebung, werden auch Datenbankmigrationen durchgeführt. Dies ist gleichzeitig eine gute Möglichkeit um sicherzugehen, dass die Migrationen mit echten Daten funktionieren.

Die Nicht-``master``-Environments sind der``master``-Environment sehr ähnlich, bis auf einige kleine Unterschiede: So werden beispielsweise E-Mails standardmäßig nicht gesendet.

.. index::
    single: Symfony CLI;open:remote

Wenn das Deployment abgeschlossen ist, öffne den neuen Branch in einem Browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Beachte, dass alle SymfonyCloud-Befehle mit dem aktuellen Git-Branch arbeiten. Somit wird der Befehl die URL für den ``sessions-in-redis``-Branch aufrufen. Die URL sieht dann so ``https://sessions-in-redis-xxx.eu.s5y.io/`` aus.

Teste die Website auf dieser neuen Environment. Du solltest jetzt alle Daten sehen, die Du in der Master-Environment angelegt hast.

Wenn Du weitere Konferenzen in die ``master``-Environment hinzufügst, werden diese nicht in der ``sessions-in-redis``-Environment angezeigt und umgekehrt. Die Environments sind unabhängig und isoliert.

Wenn sich der Code auf Master weiterentwickelt, kannst Du diese Änderungen jederzeit mittels ``rebase`` in den aktuellen Branch integrieren und die aktualisierte Version deployen, wodurch die Konflikte sowohl für den Code als auch für die Infrastruktur gelöst werden.

.. index::
    single: Symfony CLI;env:sync

Du kannst sogar die Daten von Master zurück in die ``sessions-in-redis``-Environment synchronisieren:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Fehler von Deployments in die Produktivumgebung vermeiden
---------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Standardmäßig verwenden alle SymfonyCloud-Environments die Einstellungen der ``master``/``prod``-Environment (auch bekannt als die ``prod``-Symfony-Environment). Auf diese Weise kannst Du die Anwendung unter realen Bedingungen testen. Dies gibt Dir das Gefühl, direkt auf Produktivsystemen zu entwickeln und zu testen, aber ohne den damit verbundenen Risiken. Das erinnert mich an die guten alten Zeiten, als wir Deployments noch über FTP gemacht haben.

.. index::
    single: Symfony CLI;env:debug

Wenn ein Problem auftritt, möchtest Du vielleicht auf die ``dev``-Symfony-Environment wechseln:

.. code-block:: bash

    $ symfony env:debug

Wenn Du fertig bist, gehe zurück zu den Produktiveinstellungen:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    Aktiviere **niemals** die ``dev``-Environment oder den Symfony Profiler im ``master``-Branch; dies würde Deine Anwendung wirklich langsam machen und viele ernsthafte Sicherheitsschwachstellen öffnen.

Produktivinstallationen vor dem Deployment testen
-------------------------------------------------

Der Zugriff auf die zukünftige Version der Website mit echten Daten eröffnet viele Möglichkeiten: vom visuellen Regressionstest bis zum Performance-Test. `Blackfire <https://blackfire.io>`_ ist das perfekte Werkzeug für diese Aufgabe.

Lies den Schritt über "Performance", um mehr darüber zu erfahren, wie Du Blackfire verwenden kannst, um Deinen Code vor dem Deployment zu testen.

In die Produktivumgebung mergen
-------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Wenn Du mit den Änderungen im Branch zufrieden bist, führe den Code und die Infrastruktur wieder in den Git Master-Branch zurück:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

Und deploye:

.. code-block:: bash

    $ symfony deploy

Beim Deployment werden nur die Code- und Infrastrukturänderungen in die SymfonyCloud übertragen; die Daten werden in keiner Weise beeinträchtigt.

Aufräumen
----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Entferne zum Abschluss den Git-Branch und die SymfonyCloud-Environment:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Weiterführendes

    * `Git Branching <https://www.git-scm.com/book/de/v2/Git-Branching-Branches-auf-einen-Blick>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`Stakeholder`: https://en.wikipedia.org/wiki/Project_stakeholder
