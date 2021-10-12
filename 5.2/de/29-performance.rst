Performance-Management
======================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Vorzeitige Optimierung ist die Wurzel allen Übels.

Vielleicht hast Du dieses Zitat schon einmal gelesen. Aber ich würde es gern vollständig zitieren:

.. epigraph::

    Wir sollten die kleinen Effizienzgewinne vergessen, sagen wir in 97% der Fälle: Vorzeitige Optimierung ist die Wurzel allen Übels. Dennoch sollten wir unsere Chancen bei diesen kritischen 3 % nicht verpassen.

    --   Donald Knuth

Selbst kleine Performance-Steigerungen können einen Unterschied machen, insbesondere bei E-Commerce-Websites. Nachdem die Gästebuchanwendung nun für die Prime Time bereit ist, lass uns sehen, wie wir ihre Performance überprüfen können.

Der beste Weg, Performance-Optimierungen zu finden, ist die Verwendung eines *Profilers*. Die beliebteste Option ist heutzutage `Blackfire <https://blackfire.io>`_ (*voller Haftungsausschluss*: Ich bin auch der Gründer des Blackfire-Projekts).

Blackfire
---------

Blackfire besteht aus mehreren Teilen:

* Ein *Client*, der die Analyse auslöst (das Blackfire CLI-Tool oder eine Browsererweiterung für Google Chrome oder Firefox);

* Ein *Agent*, der Daten aufbereitet und aggregiert, bevor er sie zur Anzeige an blackfire.io sendet;

* Eine PHP-Erweiterung (die * Probe*), die den PHP-Code instrumentiert.

Um mit Blackfire arbeiten zu können, musst Du Dich zuerst `anmelden <https://blackfire.io/signup>`_.

Installiere Blackfire auf Deinem lokalen Computer, indem Du das folgende Schnellinstallationsskript ausführst:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Dieser Installer lädt das Blackfire CLI Tool herunter und installiert dann die PHP-Probe (ohne sie zu aktivieren) auf allen verfügbaren PHP-Versionen.

Aktiviere die PHP-Probe für unser Projekt:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Starte den Webserver neu, damit PHP Blackfire laden kann:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Das Blackfire CLI Tool muss mit Deinen persönlichen **Client-Credentials** konfiguriert werden (um Deine Projektanalyse unter Deinem persönlichen Konto zu speichern). Du findest sie oben auf der ``Settings/Credentials`` `Seite <https://blackfire.io/my/settings/credentials>`_. Führe den folgenden Befehl aus, indem Du die Platzhalter ersetzt:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Eine vollständige Installationsanleitung findest Du in der `offiziellen ausführlichen Installationsanleitung <https://blackfire.io/docs/up-and-running/installation>`_. Sie ist nützlich, wenn Du Blackfire auf einem Server installierst.

Den Blackfire-Agenten in Docker einrichten
------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Der letzte Schritt besteht darin, den Blackfire Agent-Service in den Docker Compose-Stack aufzunehmen:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Um mit dem Server zu kommunizieren, benötigst Du Deine persönlichen **Server-Credentials** (diese identifizieren, wo Du Deine Analysen speichern möchtest – Du kannst pro Projekt eines erstellen); Du findest sie am Ende der ``Settings/Credentials``-`Seite <https://blackfire.io/my/settings/credentials>`_. Speichere sie in einer lokalen ``.env.local``-Datei:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Du kannst nun den neuen Container starten:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Eine nicht funktionierende Blackfire-Installation reparieren
------------------------------------------------------------

Wenn Du beim Analysieren einen Fehler erhälst, erhöhe das Blackfire-Log-Level, um weitere Informationen in den Logs zu erhalten:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Starte den Webserver neu:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Und lass dir die Logs ausgeben:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Analysiere erneut und überprüfe die Log-Ausgabe.


Blackfire auf dem Produktivsystem konfigurieren
-----------------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire ist standardmäßig in allen SymfonyCloud-Projekten enthalten.

Richte die *Server-Credentials* als Environment-Variable ein:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Und aktiviere die PHP-Probe wie jede andere PHP-Erweiterung:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - pdo_pgsql
             - apcu

Varnish für Blackfire konfigurieren
-----------------------------------

.. index::
    single: SymfonyCloud;Varnish

Bevor Du deployst um Blackfire nutzen zu können, benötigst Du eine Möglichkeit, den Varnish HTTP-Cache zu umgehen. Wenn nicht, wird Blackfire nie direkt auf die PHP-Anwendung zugreifen. Du wirst nur autorisierte Anfragen analysieren, die von deinem lokalen Rechner kommen.

Finde Deine aktuelle IP-Adresse heraus:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

Und verwende sie, um Varnish zu konfigurieren:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

Nun kannst Du deployen.

Websiten analysieren
--------------------

.. index::
    single: Profiling;Web Pages

Du kannst traditionelle Webseiten in Firefox oder Google Chrome über die `entsprechenden Erweiterungen <https://blackfire.io/docs/integrations/browsers/index>`_ analysieren.

Vergiss nicht, den HTTP-Cache auf Deinem lokalen Rechner in ``config/packages/framework.yaml`` beim Analysieren zu deaktivieren: Wenn nicht, wirst Du den Symfony HTTP-Cache-Layer anstelle Deines eigenen Codes analysieren:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -16,4 +16,4 @@ framework:
         php_errors:
             log: true

    -    http_cache: true
    +    #http_cache: true

Um Dir ein besseres Bild von der Performance Deiner Anwendung auf dem Produktivsystem zu machen, solltest Du auch die "Production"-Environment analysieren. Standardmäßig verwendet Deine lokale Environment die "Development"-Environment, die eine Menge Extras mit sich bringt (hauptsächlich um Daten für die Web-Debug-Toolbar und den Symfony-Profiler zu sammeln).

.. index::
    single: Symfony CLI;server:prod

Die Umstellung Deiner lokalen Maschine auf die Produktivumgebung kann durch Ändern der Environment-Variable ``APP_ENV`` in der ``.env.local``-Datei erfolgen:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Oder du kannst den ``server:prod``-Befehl verwenden:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Vergiss nicht, wieder auf dev umzustellen, wenn deine Analyse-Sitzung endet:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

API-Ressourcen analysieren
--------------------------

.. index::
    single: Profiling;API

Die Analyse der API oder der SPA erfolgt besser in der CLI über das Blackfire CLI Tool, das Du zuvor installiert hast:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Der ``blackfire curl``-Befehl akzeptiert genau die gleichen Argumente und Optionen wie `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Performancevergleich
--------------------

Im Schritt über "Cache" haben wir eine Cache-Ebene hinzugefügt, um die Performance unseres Codes zu verbessern, aber wir haben die Auswirkungen der Änderung auf die Performance weder überprüft noch gemessen. Da wir alle sehr schlecht darin sind, zu erraten, was schnell und was langsam ist, kannst Du in eine Situation geraten, in der eine Optimierung Deine Anwendung tatsächlich langsamer macht.

Du solltest immer die Auswirkungen jeder Optimierung, die Du durchführst, analysieren. Blackfire macht dies dank seiner `Vergleichsfunktion <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_ optisch einfacher.

Funktionale Black-Box-Tests schreiben
-------------------------------------

.. index::
    single: Blackfire;Player

Wir haben gesehen, wie man mit Symfony funktionale Tests schreibt. Mit Blackfire kann man Browser-Szenarien schreiben, die bei Bedarf über den `Blackfire-Player <https://blackfire.io/player>`_ ausgeführt werden können. Schreiben wir ein Szenario, das einen neuen Kommentar einreicht und ihn über den E-Mail-Link in DEV und übers Admin-Backend in PROD validiert.

Erstelle eine ``.blackfire.yaml`` Datei mit folgendem Inhalt:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Lade den Blackfire-Player herunter, um das Szenario lokal ausführen zu können:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Führe dieses Szenario in DEV aus:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Oder PROD:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Blackfire-Szenarien können auch Analysen für jeden Request auslösen und Performance-Tests durchführen, indem Du das ``--blackfire`` Flag hinzufügst.

Performance Tests automatisieren
--------------------------------

Beim Performance-Management geht es nicht nur darum, die Performance des vorhandenen Codes zu verbessern, sondern auch darum, sicherzustellen, dass keine Performanceregressionen eingeführt werden.

Das im vorherigen Abschnitt beschriebene Szenario kann automatisch in einem Continuous Integration-Workflow oder regelmäßig auf dem Produktivsystem ausgeführt werden.

In der SymfonyCloud können `die Szenarien <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ auch ausgeführt werden, wenn Du einen neuen Branch erstellst oder zum Produktivsystem deployst, um die Performance des neuen Codes automatisch zu überprüfen.

.. sidebar:: Weiterführendes

    * `Das Blackfire-Buch: PHP Code Performance erklärt <https://blackfire.io/book>`_;

    * `SymfonyCasts Blackfire Tutorial <https://symfonycasts.com/screencast/blackfire>`_.
