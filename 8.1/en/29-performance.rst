Managing Performance
====================

.. index::
    single: Blackfire
    single: Profiler

Premature optimization is the root of all evil.

Maybe you have already read this quotation before. But I like to cite it in full:

We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.

--   Donald Knuth

Even small performance improvements can make a difference, especially for e-commerce websites. Now that the guestbook application is ready for prime time, let's see how we can check its performance.

The best way to find performance optimizations is to use a *profiler*. The most popular option nowadays is `Blackfire`_ (*full disclaimer*: I am also the founder of the Blackfire project).

Introducing Blackfire
---------------------

Blackfire is made of several parts:

* A *client* that triggers profiles (the Blackfire CLI tool or a browser extension for Google Chrome or Firefox);

* An *agent* that prepares and aggregates data before sending them to blackfire.io for display;

* A PHP extension (the *probe*) that instruments the PHP code.

To work with Blackfire, you first need to `sign up`_.

Install Blackfire on your local machine by running the following quick installation script:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

This installer downloads and installs the Blackfire CLI Tool.

When done, install the PHP probe on all available PHP versions:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

And enable the PHP probe for our project:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Restart the web server so that PHP can load Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

The Blackfire CLI Tool needs to be configured with your personal **client** credentials (to store your project profiles under your personal account). Find them at the top of the ``Settings/Credentials`` `page`_ and execute the following command by replacing the placeholders:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

Setting Up the Blackfire Agent on Docker
----------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

The Blackfire agent service has already been configured in the Docker Compose stack:

.. code-block:: yaml
    :caption: compose.override.yaml
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

To communicate with the server, you need to get your personal **server** credentials (these credentials identify where you want to store the profiles -- you can create one per project); they can be found at the bottom of the ``Settings/Credentials`` `page`_. Store them as secrets:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

You can now launch the new container:

.. code-block:: terminal
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d --remove-orphans

Fixing a non-working Blackfire Installation
-------------------------------------------

If you get an error while profiling, increase the Blackfire log level to get more information in the logs:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/php.ini
    +++ w/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Restart the web server:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

And tail the logs:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Profile again and check the log output.

Configuring Blackfire in Production
-----------------------------------

.. index::
    single: Upsun;Blackfire

Blackfire is included by default in all Upsun projects.

Set up the *server* credentials as **production** secrets:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

The PHP probe is already enabled as any other needed PHP extension:

.. code-block:: yaml
    :caption: .upsun/config.yaml
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

Configuring Varnish for Blackfire
---------------------------------

.. index::
    single: Upsun;Varnish

Before you can deploy to start profiling, you need a way to bypass the Varnish HTTP cache. If not, Blackfire will never hit the PHP application. You are going to authorize the bypass of Varnish only for profiling requests coming from your local machine.

Find your current IP address:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

And use it to configure Varnish:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
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

You can now deploy.

Profiling Web Pages
-------------------

.. index::
    single: Profiling;Web Pages

You can profile traditional web pages from Firefox or Google Chrome via their `dedicated extensions`_.

On your local machine, don't forget to disable the HTTP cache in ``config/packages/framework.yaml`` when profiling: if not, you will profile the Symfony HTTP cache layer instead of your own code.

To get a better picture of the performance of your application in production, you should also profile the "production" environment. By default, your local environment is using the "development" environment, which adds a significant overhead (mainly to gather data for the web debug toolbar and the Symfony profiler).

.. note::

    As we will profile the "production" environment, there is nothing to change in the configuration as we enabled the Symfony HTTP cache layer only for the "development" environment in a previous chapter.

.. index::
    single: Symfony CLI;server:prod

Switching your local machine to the production environment can be done by changing the ``APP_ENV`` environment variable in the ``.env.local`` file:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Or you can use the ``server:prod`` command:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

Don't forget to switch it back to dev when your profiling session ends:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

Profiling API Resources
-----------------------

.. index::
    single: Profiling;API

Profiling the API is better done on the CLI via the Blackfire CLI Tool that you have installed previously:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

The ``blackfire curl`` command accepts the exact same arguments and options as `cURL`_.

Comparing Performance
---------------------

In the step about "Cache", we added a cache layer to improve the performance of our code, but we did not check nor measure the performance impact of the change. As we are all very bad at guessing what will be fast and what is slow, you might end up in a situation where making some optimization actually makes your application slower.

You should always measure the impact of any optimization you do with a profiler. Blackfire makes it visually easier thanks to its `comparison feature`_.

Writing Black Box Functional Tests
----------------------------------

.. index::
    single: Blackfire;Player

We have seen how to write functional tests with Symfony. Blackfire can be used to write browsing scenarios that can be run on demand via the `Blackfire player`_. Let's write a scenario that submits a new comment and validates it via the email link in development, and via the admin in production.

Create a ``.blackfire.yaml`` file with the following content:

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
                param comment[author] 'Fabien'
                param comment[email] 'me@example.com'
                param comment[text] 'Such a good conference!'
                param comment[photo] file(fake('simple_image', '/tmp', 400, 300, 'png', true, true), 'placeholder-image.jpg')
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

Download the Blackfire player to be able to run the scenario locally:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Run this scenario in development:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

Or in production:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

Blackfire scenarios can also trigger profiles for each request and run performance tests by adding the ``--blackfire`` flag.

Automating Performance Checks
-----------------------------

Managing performance is not only about improving the performance of existing code, it is also about checking that no performance regressions are introduced.

The scenario written in the previous section can be run automatically in a Continuous Integration workflow or in production on a regular basis.

Upsun also allows to `run the scenarios`_ whenever you create a new branch or deploy to production to check the performance of the new code automatically.

.. sidebar:: Going Further

    * `The Blackfire book: PHP Code Performance Explained`_;

    * `SymfonyCasts Blackfire tutorial`_.

.. _`Blackfire`: https://blackfire.io
.. _`sign up`: https://blackfire.io/signup
.. _`page`: https://blackfire.io/my/settings/credentials
.. _`dedicated extensions`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`comparison feature`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`Blackfire player`: https://blackfire.io/player
.. _`run the scenarios`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`The Blackfire book: PHP Code Performance Explained`: https://blackfire.io/book
.. _`SymfonyCasts Blackfire tutorial`: https://symfonycasts.com/screencast/blackfire
