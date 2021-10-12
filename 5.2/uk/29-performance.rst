Управління продуктивністю
=================================================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Передчасна оптимізація — корінь усього зла.

Можливо ви вже читали цю цитату раніше. Але я хотів би навести її повністю:

.. epigraph::

    Нам слід забути про невелику ефективність, скажімо, у 97% випадків: передчасна оптимізація — корінь усього зла. Однак ми не маємо упускати наші можливості в цих критичних 3%.

    --   Donald Knuth

Навіть невеликі поліпшення продуктивності можуть щось змінити, особливо для веб-сайтів електронної комерції. Тепер, коли застосунок гостьової книги готовий до прайм-тайму, подивімося, як ми можемо перевірити його продуктивність.

Найкращим способом пошуку потенційно слабких місць для оптимізації продуктивності є використання *профілювальника*. Найпопулярніший варіант на сьогодні — `Blackfire <https://blackfire.io>`_ (*повна відмова від відповідальності*: Я також є засновником проекту Blackfire).

Знайомство з Blackfire
---------------------------------

Blackfire складається з кількох частин:

* *Клієнт*, який запускає профілювання (інструмент CLI Blackfire або розширення для браузера Google Chrome чи Firefox);

* *Агент*, який готує й агрегує дані перед їх відправкою в blackfire.io для відображення;

* Розширення PHP (*зонд*), яке аналізує код PHP.

Щоб працювати з Blackfire, вам спочатку потрібно `зареєструватися <https://blackfire.io/signup>`_.

Встановіть Blackfire на ваш локальний комп'ютер, запустивши наступний сценарій швидкого встановлення:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Цей інсталятор завантажує інструмент CLI Blackfire, а потім встановлює PHP-зонд (не вмикаючи його) у всіх доступних версіях PHP.

Увімкніть зонд PHP для нашого проекту:

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

Перезавантажте веб-сервер, щоб PHP міг завантажити Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Інструмент CLI Blackfire має бути налаштований за допомогою ваших **клієнтських** облікових даних (щоб зберігати профілі проектів у вашому особистому кабінеті). Знайдіть їх у верхній частині `сторінки <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`` і виконайте наступну команду, замінивши заповнювачі:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Для отримання повних інструкцій із встановлення, дотримуйтеся `офіційного докладного посібника із встановлення <https://blackfire.io/docs/up-and-running/installation>`_. Вони корисні при встановленні Blackfire на сервер.

Налаштування агента Blackfire у Docker
---------------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Останнім кроком є додавання сервісу агента Blackfire в стек Docker Compose:

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

Щоб організувати обмін інформацією з сервером вам потрібно отримати ваші особисті **серверні** облікові дані (ці облікові дані визначають, де ви хочете зберігати профілі — ви можете створити один для кожного проекту); їх можна знайти в нижній частині `сторінки <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`. Збережіть їх у локальному файлі ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Тепер ви можете запустити новий контейнер:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Виправлення помилок Blackfire
-----------------------------------------------

Якщо ви отримуєте помилку під час профілювання, збільште рівень журналювання Blackfire, щоб отримувати більше інформації в журналах:

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

Перезавантажте веб-сервер:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

І стежте за журналом:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Відпрофілюйте ще раз і перевірте вивід журналу.


Налаштування Blackfire в продакшн
------------------------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire включений у всі проекти SymfonyCloud за замовчуванням.

Налаштуйте облікові дані *server* як змінні середовища:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

І увімкніть зонд PHP, як і будь-яке інше розширення PHP:

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

Налаштування Varnish для Blackfire
-------------------------------------------------

.. index::
    single: SymfonyCloud;Varnish

Перш ніж ви зможете розгорнути, щоб почати профілювання, вам потрібен спосіб, щоб обійти HTTP-кеш Varnish. Якщо ні, то Blackfire ніколи не потрапить у застосунок PHP. Ви збираєтеся дозволити обхід Varnish тільки для запитів профілювання, що надходять з вашого локального комп'ютера.

Знайдіть свою поточну IP-адресу:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

І використовуйте її для налаштування Varnish:

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

Тепер ви можете розгорнути.

Профілювання веб-сторінок
------------------------------------------------

.. index::
    single: Profiling;Web Pages

Ви можете профілювати традиційні веб-сторінки з Firefox чи Google Chrome за допомогою їх `спеціальних розширень <https://blackfire.io/docs/integrations/browsers/index>`_.

Під час профілювання не забудьте вимкнути HTTP-кеш на вашому локальному комп'ютері, у файлі ``config/packages/framework.yaml``: якщо ні, ви профілюватимите шар HTTP-кешу Symfony, замість власного коду:

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

Щоб отримати краще уявлення про продуктивність вашого застосунку в продакшн, вам також слід профілювати "production" середовище. За замовчуванням ваше локальне середовище використовує середовище "development", яке додає значні накладні витрати (в основному для збору даних для панелі інструментів веб-наладження і профілювальника Symfony).

.. index::
    single: Symfony CLI;server:prod

Перемкнути ваш локальний комп'ютер у продакшн середовище можна змінивши змінну середовища ``APP_ENV`` у файлі ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Або ви можете використовувати команду ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Не забудьте перемкнути її назад у середовище розробки, коли ваш сеанс профілювання завершиться:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Профілювання ресурсів API
---------------------------------------------

.. index::
    single: Profiling;API

Профілювання API чи ОЗ краще виконувати за допомогою CLI Blackfire, інструменту, який ви встановили раніше:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

Команда ``blackfire curl`` приймає точно ті самі аргументи й параметри, що й `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Порівняння продуктивності
-------------------------------------------------

На кроці про "Кеш" ми додали шар кешу, щоб поліпшити продуктивність нашого коду, але ми не перевіряли й не вимірювали вплив цієї зміни на продуктивність. Оскільки нам важко здогадатися, що буде швидким, а що повільним — ви можете опинитися в ситуації, коли деяка оптимізація насправді уповільнює ваш застосунок.

Ви завжди маєте вимірювати вплив будь-якої оптимізації, яку ви робите, за допомогою профілювальника. Blackfire робить це візуально простішим, завдяки своїй  `функції порівняння <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Написання функціональних тестів "чорної скриньки"
--------------------------------------------------------------------------------------------

.. index::
    single: Blackfire;Player

Ми вже бачили, як писати функціональні тести за допомогою Symfony. Blackfire можна використовувати для написання сценаріїв перегляду, які можуть бути виконані за запитом, за допомогою `Blackfire player <https://blackfire.io/player>`_. Напишімо сценарій, який відправляє новий коментар і перевіряє його за посиланням електронної пошти у середовищі розробки й за допомогою адміністратора у продакшн.

Створіть файл ``.blackfire.yaml`` із наступним вмістом:

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

Завантажте Blackfire Player, щоб мати можливість виконувати сценарій локально:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Виконайте цей сценарій у режимі розробки:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Або в продакшн:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Сценарії Blackfire також можуть профілювати кожен запит і виконувати тести продуктивності, додавши прапорець ``--blackfire``.

Автоматизація перевірок продуктивності
--------------------------------------------------------------------------

Управління продуктивністю полягає не тільки в поліпшенні продуктивності наявного коду, але і в перевірці того, що внесені зміни не призводять до її регресії.

Сценарій, що написаний у попередньому розділі, може виконуватися автоматично в робочому процесі безперервної інтеграції або у продакшн, на регулярній основі.

SymfonyCloud також дозволяє `виконувати сценарії <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ щоразу, коли ви створюєте нову гілку чи розгортаєте її у продакшн, щоб автоматично перевірити продуктивність нового коду.

.. sidebar:: Йдемо далі

    * `Книга Blackfire: Пояснення продуктивності коду PHP <https://blackfire.io/book>`_;

    * `Навчальний посібник SymfonyCasts: Blackfire <https://symfonycasts.com/screencast/blackfire>`_.
