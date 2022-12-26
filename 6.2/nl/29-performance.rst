Prestaties beheren
==================

.. index::
    single: Blackfire
    single: Profiler

Vroegtijdige optimalisaties zijn de wortel van het kwaad.

Misschien heb je dit citaat al eerder gelezen. Maar ik citeer het graag volledig:

We moeten kleine efficiëntieverbeteringen vergeten, zeg maar 97% van de tijd: voortijdige optimalisatie is de wortel van alle kwaad. Toch mogen we onze kansen in die kritische 3 procent niet voorbij laten gaan.

--   Donald Knuth

Zelfs kleine prestatieverbeteringen kunnen een verschil maken, vooral voor e-commerce websites. Nu de gastenboekapplicatie klaar is voor prime time, laten we eens kijken hoe we de prestaties ervan kunnen controleren.

De beste manier om de prestaties te optimaliseren is door het gebruik van een *profiler*. De meest populaire optie is tegenwoordig `Blackfire`_ (*volledige disclaimer*: ik ben ook de oprichter van het Blackfire project).

Introductie van Blackfire
-------------------------

Blackfire bestaat uit verschillende onderdelen:

* Een *client* die profielen activeert (de Blackfire CLI-tool of een browserextensie voor Google Chrome of Firefox);

* Een *agent* die gegevens voorbereidt en verzamelt voordat ze naar blackfire.io worden gestuurd voor weergave;

* Een PHP-extensie (de *probe*) die de PHP-code instrumenteert.

Om met Blackfire te werken, moet je eerst een `account maken`_.

Installeer Blackfire op jouw lokale machine door het volgende installatiescript uit te voeren:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

Dit installatieprogramma downloadt en installeert de Blackfire CLI Tool.

Als dit uitgevoerd wordt, installeert het de PHP probe in alle beschikbare PHP versies:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

Activeer de PHP-probe voor ons project:

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

Herstart de webserver zodat PHP Blackfire kan laden:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

De Blackfire CLI Tool moet worden geconfigureerd met jouw persoonlijke **account** gegevens (om je projectprofielen op te slaan onder je persoonlijke account). Je kan deze bovenaan de ``Settings/Credentials`` `pagina`_ vinden. Voer het volgende commando uit om de plaatsvervangers in te vullen:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

Het opzetten van de Blackfire Agent op Docker
---------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

De Blackfire agent service is reeds geconfigureerd in de Docker Compose stack:

.. code-block:: yaml
    :caption: docker-compose.override.yml
    :class: ignore

    ###> blackfireio/blackfire-symfony-meta ###
    blackfire:
        image: blackfire/blackfire:2
        # uncomment to store Blackfire credentials in a local .env.local file
        #env_file: .env.local
        environment:
        BLACKFIRE_LOG_LEVEL: 4
        ports: [8307]
    ###< blackfireio/blackfire-symfony-meta ###

Om met de server te kunnen communiceren, moet je jouw persoonlijke **server** gegevens opvragen (deze gegevens geven aan waar je de profielen wilt opslaan -- je kan er per project één aanmaken); deze kan je onderaan de ``Settings/Credentials`` `pagina`_ vinden. Bewaar ze als geheimen:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Je kan nu de nieuwe container lanceren:

.. code-block:: terminal
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Herstellen van een niet-werkende Blackfire installatie
------------------------------------------------------

Als je een fout krijgt tijdens het profileren, verhoog dan het Blackfire logniveau om meer informatie in de logs te krijgen:

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

Herstart de webserver:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

En volg de logs:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Profileer opnieuw en controleer de uitvoer van de log.

Blackfire in productie configureren
-----------------------------------

.. index::
    single: Platform.sh;Blackfire

Blackfire is standaard opgenomen in alle Platform.sh projecten.

Zet de *server* inloggegevens op als **productie** geheimen:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

De PHP probe is reeds geactiveerd, op dezelfde manier als iedere andere benodigde PHP extensie:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :emphasize-lines: 9
    :class: ignore

    runtime:
        extensions:
            - apcu
            - blackfire
            - ctype
            - iconv
            - mbstring
            - pdo_pgsql
            - sodium
            - xsl

Varnish configureren voor Blackfire
-----------------------------------

.. index::
    single: Platform.sh;Varnish

Voordat je kan starten met profileren, moet je een manier vinden om de Varnish HTTP-cache te omzeilen. Zo niet, dan zal Blackfire nooit de PHP-applicatie raken. Je wil alleen toestemming geven voor profielaanvragen die van je lokale machine komen.

Vind jouw huidige IP-adres:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

En gebruik het om Varnish te configureren:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/config.vcl
    +++ b/.platform/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "192.168.0.1";
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

Je kan nu deployen.

Webpagina's profileren
----------------------

.. index::
    single: Profiling;Web Pages

Je kan traditionele webpagina's van Firefox of Google Chrome profileren via `speciale extensies`_.

Vergeet op jouw lokale machine niet om de HTTP cache uit te schakelen in ``config/packages/framework.yaml``. Zo niet, dan zal je de Symfony HTTP cache laag in plaats van je eigen code profileren.

Om een beter beeld te krijgen van de prestaties van jouw applicatie in productie, moet je ook een profiel aanmaken van de "productieomgeving". Standaard maakt jouw lokale omgeving gebruik van de "ontwikkelomgeving", die een aanzienlijke overhead toevoegt (voornamelijk voor het verzamelen van gegevens voor de web debug toolbar en de Symfony profiler).

.. note::

    Aangezien we de "productie" omgeving zullen profileren hoeven we niets te wijzigen in de configuratie, omdat we in een vorig hoofdstuk de Symfony HTTP cache laag enkel voor de "ontwikkel" omgeving hebben geactiveerd.

.. index::
    single: Symfony CLI;server:prod

Het overschakelen van jouw lokale machine naar de productieomgeving kan worden gedaan door de ``APP_ENV`` omgevingsvariabele in het ``.env.local`` bestand te wijzigen:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Of je kunt het ``server:prod`` commando gebruiken:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

Vergeet niet om aan het einde van jouw profileringssessie terug te schakelen naar dev:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

API-resources profileren
------------------------

.. index::
    single: Profiling;API

Het profileren van de API of de SPA kan beter vanaf de CLI worden gedaan via de Blackfire CLI Tool die je eerder hebt geïnstalleerd:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Het ``blackfire curl`` commando accepteert exact dezelfde argumenten en opties als `cURL`_.

Prestaties vergelijken
----------------------

In de stap over "Cache" hebben we een cache-laag toegevoegd om de prestaties van onze code te verbeteren, maar we hebben de impact van de verandering op de prestaties niet gecontroleerd of gemeten. Omdat je slecht kan raden of een aanpassing alles sneller of net trager maakt, kan het zijn dat je fout raad en je jouw applicatie juist langzamer maakt in plaats van sneller.

Je zou altijd de impact van elke uitgevoerde optimalisatie moeten meten met een profiler. Blackfire maakt dit visueel gemakkelijker dankzij de `vergelijkingsfunctie`_.

Het schrijven van functionele blackboxtesten
--------------------------------------------

.. index::
    single: Blackfire;Player

We hebben gezien hoe we functionele tests kunnen schrijven met Symfony. Blackfire kan gebruikt worden om surf-scenario's te schrijven die op verzoek via de `Blackfire-player`_ kunnen worden uitgevoerd. Laten we een scenario schrijven dat een nieuw commentaar indient en deze valideert via de e-mail link in development en via de admin in productie.

Maak een ``.blackfire.yaml`` bestand aan met de volgende inhoud:

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
                param comment_form[photo] file(fake('simple_image', '/tmp', 400, 300, 'png', true, true), 'placeholder-image.jpg')
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
                visit url('/admin')
                    expect status_code() == 302
                follow
                click link("Comments")
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

Download de Blackfire-player om het scenario lokaal te kunnen uitvoeren:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar
    $ cp /home/fabien/Code/github/blackfireio/blackfire.io/player/blackfire-player.phar blackfire-player.phar

Voer dit scenario uit in development:

.. code-block:: terminal

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

Of in productie:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

Blackfire-scenario's kunnen ook profielen triggeren voor elke request en prestatietests uitvoeren door het toevoegen van de ``--blackfire`` parameter.

Automatiseren van performancecontroles
--------------------------------------

Het managen van de prestaties is niet alleen het verbeteren van de prestaties van bestaande code, maar ook het controleren of er geen prestatie-regressies worden geïntroduceerd.

Het scenario dat in de vorige paragraaf is beschreven, kan automatisch worden uitgevoerd in een Continuous Integration-workflow of op regelmatige basis in productie.

Platform.sh maakt het mogelijk om `de scenario's uit te voeren`_ wanneer je een nieuwe branch maakt of deployed naar productie om de prestaties van de nieuwe code automatisch te controleren.

.. sidebar:: Verder gaan

    * `Het Blackfire boek: PHP Code Performance Explained`_;

    * `SymfonyCasts Blackfire tutorial`_.

.. _`Blackfire`: https://blackfire.io
.. _`account maken`: https://blackfire.io/signup
.. _`pagina`: https://blackfire.io/my/settings/credentials
.. _`speciale extensies`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`vergelijkingsfunctie`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`Blackfire-player`: https://blackfire.io/player
.. _`de scenario's uit te voeren`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`Het Blackfire boek: PHP Code Performance Explained`: https://blackfire.io/book
.. _`SymfonyCasts Blackfire tutorial`: https://symfonycasts.com/screencast/blackfire
