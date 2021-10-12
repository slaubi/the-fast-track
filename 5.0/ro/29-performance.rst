Gestionarea performanței
=========================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Optimizarea prematură este rădăcina tuturor răurilor.

Poate că ați citit deja acest citat. Dar îmi place să-l citez în întregime:

.. epigraph::

    Ar trebui să uităm de eficiențe mici, să spunem că aproximativ 97% din timp: optimizarea prematură este rădăcina tuturor răurilor. Cu toate acestea, nu ar trebui să ne ferim de oportunitățile celor 3% critice.

    --   Donald Knuth

Chiar și îmbunătățiri mici de performanță pot face diferența, în special pentru site-urile de comerț electronic. Acum că aplicația pentru cartea de oaspeți este pregătită pentru lansare, hai să vedem cum putem verifica performanța acesteia.

Cea mai bună modalitate de a găsi optimizări ale performanței este să utilizezi un *profilator*. Cea mai populară opțiune în zilele noastre este `Blackfire <https://blackfire.io>`_ (*disclaimer complet*: Sunt și fondatorul proiectului Blackfire).

Prezentarea Blackfire
---------------------

Blackfire este format din mai multe componente:

* Un *client* care declanșează profiluri (instrumentul CLI Blackfire sau o extensie de browser pentru Google Chrome sau Firefox);

* Un *agent* care pregătește și agreghează date înainte de a le expedia la blackfire.io pentru afișare;

* O extensie PHP (*sonda*) care instrumentează codul PHP.

Pentru a lucra cu Blackfire, trebuie mai întâi să îți `creezi un cont Blackfire <https://blackfire.io/signup>`_

Instalează Blackfire pe mașina ta locală rulând următorul script de instalare rapidă:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Acest instalator descarcă Blackfire CLI Tool și apoi instalează sonda PHP (fără a o activa) pe toate versiunile PHP disponibile.

Activează sonda PHP pentru proiectul nostru:

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

Repornește serverul web pentru ca PHP să poată încărca Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Instrumentul CLI Blackfire trebuie configurat cu datele tale personale **client** (pentru stocarea profilurilor proiectului în contul personal). Găsește-le în partea de sus a `paginii <https://blackfire.io/my/settings/credentials>`_ ``Setări/Credențiale`` și execută următoarea comandă prin înlocuirea marcatorilor:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Pentru instrucțiuni complete de instalare, urmează `ghidul de instalare oficial detaliat <https://blackfire.io/docs/up-and-running/installation>`_. Sunt utile când instalezi Blackfire pe un server.

Configurarea agentului Blackfire pe Docker
------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Ultimul pas este adăugarea serviciului agentului Blackfire în stack-ul Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -20,3 +20,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Pentru a comunica cu serverul, trebuie să obții datele tale personale de identificare pentru **server** (aceste date identifică locul în care dorești să stochezi profilurile - poți crea unul per proiect); acestea pot fi găsite în partea de jos a `paginii <https://blackfire.io/my/settings/credentials>`_ ``Setări/Credențiale``. Stochează-le într-un fișier local ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Acum poți lansa noul container:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Repararea unei instalări Blackfire care nu funcționează
----------------------------------------------------------

Dacă obții o eroare în timpul profilării, marește nivelul de raportare al erorilor în Blackfire pentru a obține mai multe informații în jurnale:

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

Repornește serverul web:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Și apoi listează ultimele mesaje din jurnal:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Profilează din nou și verifică jurnalul.


Configurarea Blackfire în producție
-------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire este inclus în mod implicit în toate proiectele SymfonyCloud.

Configură datele de autentificare *server* ca variabile de mediu:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Și activează sonda PHP ca orice altă extensie PHP:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - amqp
             - redis

Configurarea Varnish pentru Blackfire
-------------------------------------

.. index::
    single: SymfonyCloud;Varnish

Înainte de a putea începe profilarea, ai nevoie de o modalitate de a ocoli memoria cache Varnish HTTP. Dacă nu, Blackfire nu va atinge niciodată aplicația PHP. Vei autoriza doar cererile de profil provenite de la mașina locală.

Găsește adresa ta IP curentă:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

Și utilizeaz-o pentru a configura Varnish:

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

Acum poți lansa în producție.

Profilarea paginilor web
------------------------

.. index::
    single: Profiling;Web Pages

Poți profila paginile web tradiționale de pe Firefox sau Google Chrome prin `extensiile lor dedicate <https://blackfire.io/docs/integrations/browsers/index>`_.

Pe mașina locală, nu uita să dezactivezi memoria cache HTTP în ``public/index.php`` atunci când profilezi: dacă nu, vei profila stratul cache HTTP Symfony în loc de propriul cod:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/public/index.php
    +++ b/public/index.php
    @@ -24,7 +24,7 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? $_ENV['TRUSTED_HOSTS'] ?? false
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);

     if ('dev' === $kernel->getEnvironment()) {
    -    $kernel = new HttpCache($kernel);
    +//    $kernel = new HttpCache($kernel);
     }

     $request = Request::createFromGlobals();

Pentru a obține o imagine mai bună a performanței aplicației  în producție, ar trebui să profilezi și mediul „de producție”. În mod implicit, mediul local folosește mediul „dezvoltare”, care adaugă o penalitate semnificativă de performanță (în principal pentru a aduna date pentru bara de instrumente de depanare web și profilatorul Symfony).

.. index::
    single: Symfony CLI;server:prod

Transferul mașinei locale la mediul de producție se poate face modificând variabila de mediu ``APP_ENV`` din fișierul ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Sau poți utiliza comanda ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Nu uita să-l schimbi înapoi la dev când se încheie sesiunea de profilare:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Profilarea resurselor API
-------------------------

.. index::
    single: Profiling;API

Profilarea API-ului sau SPA-ului se realizează mai bine din linia de comandă prin intermediul instrumentului Blackfire CLI pe care l-ai instalat anterior:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Comanda ``blackfire curl`` acceptă aceleași argumente și opțiuni precum `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Compararea performanțelor
--------------------------

În pasul despre „Cache”, am adăugat un strat de cache pentru a îmbunătăți performanța codului nostru, dar nu am verificat și nici nu am măsurat impactul asupra performanței modificării. Întrucât nu suntem capabili să ghicim ce va fi rapid și ce este lent, s-ar putea să ajungem într-o situație în care realizarea unor optimizări face efectiv aplicația mai lentă.

Ar trebui să măsurăm întotdeauna impactul oricărei optimizări pe care le efectuăm cu un profilator. Blackfire ușurează sarcina din punct de vedere vizual datorită `funcției sale de comparare <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Scrierea testelor funcționale Black Box
----------------------------------------

.. index::
    single: Blackfire;Player

Am văzut cum să scriem teste funcționale cu Symfony. Blackfire poate fi utilizat pentru a scrie scenarii de navigare care pot fi rulate la cerere prin intermediul `Blackfire player <https://blackfire.io/player>`_. Să scriem un scenariu care expediază un nou comentariu și îl validează prin intermediul legăturii de e-mail în dezvoltare și prin intermediul administratorului în producție.

Creează un fișier ``.blackfire.yaml`` cu următorul conținut:

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

Descarcă playerul Blackfire pentru a putea rula scenariul local:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Execută acest scenariu în mediul dev:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Sau în producție:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Scenariile Blackfire pot, de asemenea, să declanșeze profiluri pentru fiecare solicitare și să ruleze teste de performanță adăugând opțiunea ``--blackfire``.

Automatizarea verificărilor de performanță
---------------------------------------------

Gestionarea performanței nu înseamnă doar îmbunătățirea performanței codului existent, ci și verificarea dacă nu sunt introduse regresii de performanță.

Scenariul scris în secțiunea anterioară poate fi rulat automat într-un flux de lucru cu integrare continuă sau în producție în mod regulat.

SymfonyCloud, de asemenea permite `rularea scenariilor <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_  automat ori de câte ori creăm o ramură nouă sau lansăm în producție pentru a verifica performanța.

.. sidebar:: Mergând mai departe

    * `Cartea Blackfire: Performanța codului PHP explicată <https://blackfire.io/book>`_;

    * `Tutorial SymfonyCasts Blackfire <https://symfonycasts.com/screencast/blackfire>`_.
