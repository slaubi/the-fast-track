Оптимизация производительности
===========================================================

.. index::
    single: Blackfire
    single: Profiler

Преждевременная оптимизация — корень всех зол в программировании.

Возможно вы встречали эту цитату и ранее, однако я люблю цитировать её полностью:

Нам следует забывать о небольшой эффективности, например, в 97% случаев: преждевременная оптимизация — корень всех зол. И напротив, мы должны уделить всё внимание оставшимся 3%.

--   Donald Knuth

Даже незначительное увеличение производительности имеет важное значение, особенно для интернет-магазинов. Теперь, когда приложение гостевой книги готово, давайте проверим его производительность.

Лучший способ определить потенциальные области для оптимизации — использовать *профилировщик*. Одним из наиболее популярных на сегодняшний день является `Blackfire`_ (*важное примечание*: я также основатель проекта Blackfire).

Знакомство с Blackfire
---------------------------------

Blackfire состоит из нескольких частей:

* *Клиент*, запускающий профили (CLI-инструмент Blackfire или расширение для браузера Google Chrome или Firefox);

* *Агент*, который подготавливает и собирает данные перед их отправкой на сайт blackfire.io для отображения;

* PHP-модуль (*зонд*), который инструментирует PHP-код.

`Зарегистрируйтесь`_, чтобы начать работу с Blackfire.

Установите Blackfire на вашу локальную машину, запустив скрипт быстрой установки:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

Установщик загрузит и установит CLI-инструмент Blackfire.

По завершении устанавливает PHP-зонд на все доступные версии PHP:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

И включите PHP-зонд для нашего проекта:

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

Перезапустите веб-сервер, чтобы PHP cмог загрузить Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Для хранения профилей приложений в вашей учётной записи, необходимо настроить инструмент Blackfire CLI, используя ваши персональные **клиентские** учётные данные. Найдите их в верху `страницы`_ ``Settings/Credentials`` и выполните следующую команду, предварительно подставив свои данные в соответствующие места:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

Установка Blackfire-агента в Docker
---------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Служба агента Blackfire уже настроена в стеке Docker Compose:

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

Для соединения с сервером вам необходимо получить ваши персональные **серверные** учётные данные. Эти учётные данные указывают, где вы хотите хранить профили, которые вы можете создать для каждого проекта. Их можно найти внизу `страницы`_ ``Settings/Credentials``. Сохраните данные локально в файле ``.env.local``:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Теперь запустите новый контейнер:

.. code-block:: terminal
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Исправление неработающей установки Blackfire
----------------------------------------------------------------------------

Если во время профилирования появляется ошибка — увеличьте уровень логирования ошибок Blackfire, чтобы собирать больше информации в отчётах об ошибках:

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

Перезапустите веб-сервер:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

И следите за логами:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Запустите профилирование снова и проверьте записи лога.

Настройка Blackfire в продакшене
----------------------------------------------------

.. index::
    single: Platform.sh;Blackfire

По умолчанию Blackfire добавлен во все проекты на Platform.sh.

Установите учетные данные *сервера* в качестве секретов для окружения **production**:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Зонд PHP уже включён, как и любой другой необходимый модуль PHP:

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

Настройка Varnish для работы с Blackfire
-----------------------------------------------------------

.. index::
    single: Platform.sh;Varnish

Перед развёртыванием, чтобы приступить к профилированию, вам необходимо обойти HTTP-кеш Varnish. Если этого не сделать — Blackfire никогда не сможет получить доступ к вашему PHP-приложению. Достаточно разрешить HTTP-запросы профилирования, приходящие только с вашего локального компьютера.

Узнайте ваш текущий IP-адрес:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

И используйте его для настройки Varnish:

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

Теперь можно разворачивать.

Профилирование веб-страниц
--------------------------------------------------

.. index::
    single: Profiling;Web Pages

Для профилирования обычных веб-страниц в Firefox или Google Chrome используйте `соответствующие расширения`_.

Во время профилирования не забудьте отключить HTTP-кеш на вашей локальной машине в файле ``config/packages/framework.yaml``. Если этого не выполнить, то вместо своего кода, вы будете профилировать слой HTTP-кеша Symfony.

Для получения наиболее полного представления о производительности приложения в продакшене, вам также необходимо профилировать окружение продакшена ("production"). По умолчанию ваше локальное окружение использует среду разработки ("development"), что влечёт за собой существенные накладные расходы (это происходит в основном из-за сбора данных для отладочной панели и Symfony-профилировщика).

.. note::

    Поскольку мы будем профилировать окружение "production", в конфигурации ничего менять не нужно, так как в предыдущей главе мы включили слой HTTP-кеша Symfony только для среды "development".

.. index::
    single: Symfony CLI;server:prod

Переключите окружение вашей локальной машины на работу в продакшене путём изменения значения переменной окружения  ``APP_ENV`` в файле ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Также вы можете использовать команду ``server:prod``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

Не забудьте переключиться обратно на окружение разработки после завершения сеанса профилирования:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

Профилирование API-ресурсов
-------------------------------------------------

.. index::
    single: Profiling;API

Профилирование API или SPA лучше выполнять в командной строке с помощью установленного ранее инструмента Blackfire CLI:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Команда ``blackfire curl`` принимает те же аргументы и опции, что и `cURL`_.

Сравнение производительности
-------------------------------------------------------

В шаге про кеширование для повышения производительности мы добавили слой кеширования, однако мы не проверили, как внесённые изменения повлияли на производительность. Так как довольно сложно угадать, что будет работать быстрее, а что медленнее, то можно оказаться в ситуации, когда из-за  оптимизации чего-либо ваше приложение на самом деле станет только медленнее.

С помощью профилировщика всегда необходимо измерять влияние каждой сделанной оптимизации. Blackfire позволяет упростить визуальную оценку производительности благодаря `возможности сравнения`_.

Написание функциональных тестов по принципу чёрного ящика
------------------------------------------------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Мы уже знакомы с тем, как писать функциональные тесты с помощью Symfony. Blackfire, в свою очередь, может быть использован для написания браузерных сценариев, которые можно запускать по желанию с помощью приложения `Blackfire player`_. Давайте напишем сценарий, который отправит новый комментарий и проверит его по ссылке в электронной почте в окружении для разработки, а также в административной панели в продакшене.

Создайте файл ``.blackfire.yaml`` со следующим содержимым:

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

Загрузите Blackfire player для выполнения сценария на локальной машине:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar
    $ cp /home/fabien/Code/github/blackfireio/blackfire.io/player/blackfire-player.phar blackfire-player.phar

Запустите этот сценарий в окружении разработки:

.. code-block:: terminal

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

Или в продакшене:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

Сценарии Blackfire, также могут использовать профили для каждого запроса и запускать тесты производительности, если добавить соответствующий флаг ``--blackfire``.

Автоматизация проверок производительности
--------------------------------------------------------------------------------

Отслеживание производительности позволяет понять не только как улучшить производительность существующего кода, но также и контролировать её падение, вызванное новыми изменениями.

Написанный в предыдущем разделе сценарий может запускаться автоматически через непрерывную интеграцию или на регулярной основе в продакшене.

На Platform.sh также возможно `запускать сценарии`_ при создании новой ветки или развёртывании в продакшене, чтобы автоматически проверять производительность нового кода.

.. sidebar:: Двигаемся дальше

    * `Книга Blackfire: объяснение производительности PHP-кода`_;

    * `Обучающий видеокурс по Blackfire на SymfonyCasts`_.

.. _`Blackfire`: https://blackfire.io
.. _`Зарегистрируйтесь`: https://blackfire.io/signup
.. _`страницы`: https://blackfire.io/my/settings/credentials
.. _`соответствующие расширения`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`возможности сравнения`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`Blackfire player`: https://blackfire.io/player
.. _`запускать сценарии`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`Книга Blackfire: объяснение производительности PHP-кода`: https://blackfire.io/book
.. _`Обучающий видеокурс по Blackfire на SymfonyCasts`: https://symfonycasts.com/screencast/blackfire
