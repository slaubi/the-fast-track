Оптимизация производительности
===========================================================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Преждевременная оптимизация — корень всех зол в программировании.

Возможно вы встречали эту цитату и ранее, однако я люблю цитировать её полностью:

.. epigraph::

    Нам следует забывать о небольшой эффективности, например, в 97% случаев: преждевременная оптимизация — корень всех зол. И напротив, мы должны уделить всё внимание оставшимся 3%.

    --   Donald Knuth

Даже незначительное увеличение производительности имеет важное значение, особенно для интернет-магазинов. Теперь, когда приложение гостевой книги готово, давайте проверим его производительность.

Лучший способ определить потенциальные области для оптимизации — использовать *профилировщик*. Одним из наиболее популярных на сегодняшний день является `Blackfire <https://blackfire.io>`_ (*важное примечание*: я также основатель проекта Blackfire).

Знакомство с Blackfire
---------------------------------

Blackfire состоит из нескольких частей:

* *Клиент*, запускающий профили (CLI-инструмент Blackfire или расширение для браузера Google Chrome или Firefox);

* *Агент*, который подготавливает и собирает данные перед их отправкой на сайт blackfire.io для отображения;

* PHP-модуль (*зонд*), который инструментирует PHP-код.

`Зарегистрируйтесь <https://blackfire.io/signup>`_, чтобы начать работу с Blackfire.

Установите Blackfire на вашу локальную машину, запустив скрипт быстрой установки:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Установщик загрузит CLI-инструмент Blackfire, а затем установит PHP-модуль зонда (не активируя его) для всех доступных версий PHP.

Включите PHP-зонд для нашего проекта:

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

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Для хранения профилей приложений в вашей учётной записи, необходимо настроить инструмент Blackfire CLI, используя ваши персональные **клиентские** учётные данные. Найдите их в верху `страницы <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`` и выполните следующую команду, предварительно подставив свои данные в соответствующие места:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Полную инструкцию по установке вы сможете найти в `официальном подробном руководстве по установке <https://blackfire.io/docs/up-and-running/installation>`_. Она также полезна при установке Blackfire на сервер.

Установка Blackfire-агента в Docker
---------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Последним шагом будет добавление сервиса Blackfire-агента в стек Docker Compose:

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

Для соединения с сервером вам необходимо получить ваши персональные **серверные** учётные данные. Эти учётные данные указывают, где вы хотите хранить профили, которые вы можете создать для каждого проекта. Их можно найти внизу `страницы <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials``. Сохраните данные локально в файле ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Теперь запустите новый контейнер:

.. code-block:: bash
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

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

И следите за логами:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Запустите профилирование снова и проверьте записи лога.


Настройка Blackfire в продакшене
----------------------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

По умолчанию Blackfire добавлен во все проекты на SymfonyCloud.

Сохраните *серверные* учётные данные в переменных окружения:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

И активируйте PHP-модуль зонда по аналогии с любым другим PHP-модулем:

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

Настройка Varnish для работы с Blackfire
-----------------------------------------------------------

.. index::
    single: SymfonyCloud;Varnish

Перед развёртыванием, чтобы приступить к профилированию, вам необходимо обойти HTTP-кеш Varnish. Если этого не сделать — Blackfire никогда не сможет получить доступ к вашему PHP-приложению. Достаточно разрешить HTTP-запросы профилирования, приходящие только с вашего локального компьютера.

Узнайте ваш текущий IP-адрес:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

И используйте его для настройки Varnish:

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

Теперь можно разворачивать.

Профилирование веб-страниц
--------------------------------------------------

.. index::
    single: Profiling;Web Pages

Для профилирования обычных веб-страниц в Firefox или Google Chrome используйте `соответствующие расширения <https://blackfire.io/docs/integrations/browsers/index>`_.

Во время профилирования не забудьте отключить HTTP-кеш на вашей локальной машине в файле ``config/packages/framework.yaml``. Если этого не выполнить, то вместо своего кода, вы будете профилировать слой HTTP-кеша Symfony.

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

Для получения наиболее полного представления о производительности приложения в продакшене, вам также необходимо профилировать окружение продакшена ("production"). По умолчанию ваше локальное окружение использует среду разработки ("development"), что влечёт за собой существенные накладные расходы (это происходит в основном из-за сбора данных для отладочной панели и Symfony-профилировщика).

.. index::
    single: Symfony CLI;server:prod

Переключите окружение вашей локальной машины на работу в продакшене путём изменения значения переменной окружения  ``APP_ENV`` в файле ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Также вы можете использовать команду ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Не забудьте переключиться обратно на окружение разработки после завершения сеанса профилирования:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Профилирование API-ресурсов
-------------------------------------------------

.. index::
    single: Profiling;API

Профилирование API или SPA лучше выполнять в командной строке с помощью установленного ранее инструмента Blackfire CLI:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Команда ``blackfire curl`` принимает те же аргументы и опции, что и `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Сравнение производительности
-------------------------------------------------------

В шаге про кеширование для повышения производительности мы добавили слой кеширования, однако мы не проверили, как внесённые изменения повлияли на производительность. Так как довольно сложно угадать, что будет работать быстрее, а что медленнее, то можно оказаться в ситуации, когда из-за  оптимизации чего-либо ваше приложение на самом деле станет только медленнее.

С помощью профилировщика всегда необходимо измерять влияние каждой сделанной оптимизации. Blackfire позволяет упростить визуальную оценку производительности благодаря `возможности сравнения <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Написание функциональных тестов по принципу чёрного ящика
------------------------------------------------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Мы уже знакомы с тем, как писать функциональные тесты с помощью Symfony. Blackfire, в свою очередь, может быть использован для написания браузерных сценариев, которые можно запускать по желанию с помощью приложения `Blackfire player <https://blackfire.io/player>`_. Давайте напишем сценарий, который отправит новый комментарий и проверит его по ссылке в электронной почте в окружении для разработки, а также в административной панели в продакшене.

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

Загрузите Blackfire player для выполнения сценария на локальной машине:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Запустите этот сценарий в окружении разработки:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Или в продакшене:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Сценарии Blackfire, также могут использовать профили для каждого запроса и запускать тесты производительности, если добавить соответствующий флаг ``--blackfire``.

Автоматизация проверок производительности
--------------------------------------------------------------------------------

Отслеживание производительности позволяет понять не только как улучшить производительность существующего кода, но также и контролировать её падение, вызванное новыми изменениями.

Написанный в предыдущем разделе сценарий может запускаться автоматически через непрерывную интеграцию или на регулярной основе в продакшене.

На SymfonyCloud также возможно `запускать сценарии <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>` при создании новой ветки или развёртывании в продакшене, чтобы автоматически проверять производительность нового кода.

.. sidebar:: Двигаемся дальше

    * `Книга Blackfire: объяснение производительности PHP-кода <https://blackfire.io/book>`_;

    * `Обучающий видеокурс по Blackfire на SymfonyCasts <https://symfonycasts.com/screencast/blackfire>`_.
