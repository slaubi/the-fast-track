Gestire le prestazioni
======================

.. index::
    single: Blackfire
    single: Profiler

L'ottimizzazione prematura è la radice di tutti i mali.

Forse avrete già letto questa citazione in precedenza, ma vorrei riportarla per intero:

Si dovrebbero tralasciare le micro-ottimizzazioni, diciamo circa il 97% delle volte: l'ottimizzazione prematura è la radice di tutti i mali. Tuttavia, non dovremmo tralasciare l'opportunità di quel 3% critico.

--   Donald Knuth

Anche piccoli miglioramenti delle prestazioni possono fare la differenza, specialmente per i siti di e-commerce. Ora che l'applicazione guestbook è pronta per il debutto, vediamo come possiamo verficarne le prestazioni.

Il modo migliore per scovare quali ottimizzazioni sulle prestazioni potremmo fare è quello di utilizzare un *profiler*. L'opzione più popolare al giorno d'oggi è `Blackfire`_ (*disclaimer* : sono anche il fondatore del progetto Blackfire).

Introduzione a Blackfire
------------------------

Blackfire è composto da più parti:

* Un *client* che attiva i profili (lo strumento CLI di Blackfire o un'estensione del browser per Google Chrome o Firefox);

* Un *agent* che prepara e aggrega i dati prima di inviarli a blackfire.io per la visualizzazione;

* Un'estensione PHP (detta *probe*) che fornisce istruzioni al codice PHP.

Per lavorare con Blackfire è necessario `iscriversi`_.

Installare Blackfire sul computer locale eseguendo il seguente script di installazione rapida:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

Questo programma di installazione scarica ed installa la CLI di Blackfire.

Una volta fatto, installiamo l'estensione PHP probe su tutte le versioni PHP disponibili:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

E attiva l'estensione probe di PHP per il progetto:

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

Riavviare il server web in modo che PHP possa caricare Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

La CLI di Blackfire va configurata inserendo le credenziali del **client** (per memorizzare i profili dei progetti sotto il proprio account). Si trovano nella parte superiore della `pagina`_ ``Settings/Credentials``. Eseguire quindi il seguente comando, sostituendo i segnaposto:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

Impostazione dell'agent di Blackfire su Docker
----------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Il servizio agent di Blackfire è già stato configurato nello stack di Docker Compose:

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

Per comunicare con il server, è necessario ottenere le credenziali del **server** (queste credenziali indicano dove si desidera memorizzare i profili e possono essere personalizzate per singolo progetto); si trovano in fondo alla `pagina`_ ``Settings/Credentials``. Memorizzarle in un file locale ``.env.local``:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

A questo punto è possibile lanciare il nuovo container:

.. code-block:: terminal
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Sistemare un'installazione di Blackfire non funzionante
-------------------------------------------------------

Se si riscontra un errore durante la profilazione, aumentare il livello dei log di Blackfire per ottenere maggiori informazioni:

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

Riavviare il server web:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Ed eseguire un tail sui log:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Eseguire nuovamente la profilazione e controllare l'output del log.

Configurazione di Blackfire in produzione
-----------------------------------------

.. index::
    single: Platform.sh;Blackfire

Blackfire è incluso di default in tutti i progetti Platform.sh.

Imposta le credenziali del *server* come segreti di **production**:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

L'estensione PHP probe è già abilitata come ogni altra estensione PHP necessaria:

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

Configurazione di Varnish per Blackfire
---------------------------------------

.. index::
    single: Platform.sh;Varnish

Prima di poter eseguire il deploy per iniziare la profilazione, è necessario un modo per aggirare la cache HTTP di Varnish. In caso contrario, Blackfire non arriverà mai all'applicazione PHP. Autorizzeremo solo le richieste di profilazione provenienti dalla macchina locale.

Trovare il proprio indirizzo IP:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

E usarlo per configurare Varnish:

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

Ora è possibile eseguire il deploy.

Profilazione delle pagine web
-----------------------------

.. index::
    single: Profiling;Web Pages

È possibile profilare le pagine web tradizionali da Firefox o da Google Chrome tramite le rispettive `estensioni dedicate`_.

Sulla macchina locale, non dimenticare di disabilitare la cache HTTP in ``config/packages/framework.yaml`` durante la profilazione: in caso contrario, verrà profilato il livello di cache HTTP di Symfony invece del proprio codice.

Per ottenere un quadro migliore delle prestazioni della propria applicazione in produzione, si dovrebbe profilare anche l'ambiente "production". Per impostazione predefinita, l'ambiente locale utilizza l'ambiente "development", che aggiunge un overhead significativo (principalmente per raccogliere dati per la barra degli strumenti di debug e il Profiler di Symfony).

.. note::

    Quando profileremo l'ambiente "production", non ci sarà nessuna configurazione da cambiare, visto che abbiamo abilitato il layer di cache HTTP solo per l'ambiente "development" in un capitolo precedente.

.. index::
    single: Symfony CLI;server:prod

Il passaggio dalla macchina locale all'ambiente di produzione può essere fatto cambiando la variabile d'ambiente ``APP_ENV`` nel file ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Oppure si può usare il comando ``server:prod``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

Non dimentichiamo di tornare in dev quando la sessione di profilazione è terminata:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

Profilazione delle risorse API
------------------------------

.. index::
    single: Profiling;API

La profilazione delle API o della SPA viene fatta in modo migliore sul terminale tramite la CLI di Blackfire, installata in precedenza:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Il comando ``blackfire curl`` accetta esattamente gli stessi parametri e opzioni di `cURL`_.

Confronto delle prestazioni
---------------------------

Nel passo sulla "Cache", abbiamo aggiunto uno strato di cache per migliorare le prestazioni del codice, ma non abbiamo controllato né misurato l'impatto sulle prestazioni di tale modifica. Dato che non siamo molto bravi nell'indovinare cosa sarà veloce e cosa è lento, potremmo ritrovarci in una situazione in cui eseguire qualche ottimizzazione rende l'applicazione più lenta.

Si dovrebbe sempre misurare l'impatto di ogni ottimizzazione, usando un profiler. Blackfire rende il compito visivamente più facile grazie alla sua `funzionalità di confronto`_.

Scrivere test funzionali di tipo Black Box
------------------------------------------

.. index::
    single: Blackfire;Player

Abbiamo visto come scrivere test funzionali con Symfony. Blackfire può essere utilizzato per scrivere scenari di navigazione che possono essere eseguiti su richiesta tramite il `player di Blackfire`_. Scriviamo uno scenario che invia un nuovo commento e lo convalida tramite un link via e-mail in sviluppo e tramite un'interfaccia di amministrazione.

Creare un file ``.blackfire.yaml`` con il seguente contenuto:

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

Scarichiamo il player di Blackfire per poter eseguire lo scenario in locale:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar
    $ cp /home/fabien/Code/github/blackfireio/blackfire.io/player/blackfire-player.phar blackfire-player.phar

Eseguire questo scenario in sviluppo:

.. code-block:: terminal

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

O in produzione:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

Gli scenari di Blackfire possono anche attivare profili per ogni richiesta ed eseguire test sulle prestazioni aggiungendo l'opzione ``--blackfire``.

Automatizzare i controlli delle prestazioni
-------------------------------------------

Gestire le prestazioni non significa solo migliorare le prestazioni del codice esistente, ma anche verificare che non vengano introdotte regressioni.

Lo scenario descritto nella sezione precedente può essere eseguito automaticamente, su base regolare, in una attività di CI (Continuous Integration) o in produzione.

Su Platform.sh, consente anche `eseguire gli scenari`_ ogni volta che si crea un nuovo branch o si fa un deploy in produzione, per controllare automaticamente le prestazioni del nuovo codice.

.. sidebar:: Andare oltre

    * `Il libro di Blackfire: PHP Code Performance Explained`_;

    * `Guida a Blackfire su SymfonyCasts`_.

.. _`Blackfire`: https://blackfire.io
.. _`iscriversi`: https://blackfire.io/signup
.. _`pagina`: https://blackfire.io/my/settings/credentials
.. _`estensioni dedicate`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`funzionalità di confronto`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`player di Blackfire`: https://blackfire.io/player
.. _`eseguire gli scenari`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`Il libro di Blackfire: PHP Code Performance Explained`: https://blackfire.io/book
.. _`Guida a Blackfire su SymfonyCasts`: https://symfonycasts.com/screencast/blackfire
