مدیریت کارایی
=========================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    بهینه‌سازی قبل از موعد، ریشه‌ی تمام بدبختی‌ها است!

شاید شما قبلاً این نقل قول را خوانده باشید. اما دوست دارم آن را به طور کامل ذکر کنم:

.. epigraph::

    ما باید بهبود‌های جزئی را فراموش کنیم، می‌توان در ۹۷ درصد موارد گفت: بهینه‌سازی قبل از موعد، ریشه‌ی تمام بدبختی‌ها است. البته نباید از فرصت‌های موجود در آن ۳ درصد حیاتی صرف نظر کنیم.

    --   Donald Knuth

هر بهبود کارایی جزئی، به خصوص برای وب‌سایت‌های تجارت الکترونیک (e-commerce) می‌تواند تفاوت ایجاد کند. حالا اپلیکیشن guestbook برای نسختین بار آماده است، بیایید ببینیم چگونه می‌توان کارایی‌ آن را بررسی کرد.

بهترین راه برای پیدا کردن بهینه سازی‌های کارایی، استفاده از یک *نمایه‌ساز (profiler)* است. امروزه محبوب‌ترین آن‌ها `Blackfire <https://blackfire.io>`_ است (سلب مسئولیت تام: من بنیان‌گذار پروژه‌ی Blackfire نیز هستم).

معرفی Blackfire
--------------------

Blackfire از بخش‌های متعددی ساخته شده است:

* یک *client* که نمایه‌سازی را راه می‌اندازد (ابزار CLI مربوط به Blackfire یا یک افزونه‌ی مرورگر برای گوگل کروم یا فایرفاکس)؛

* یک *مأمور (agent)* که داده‌ها را برای ارسال به blackfire.io جهت نمایش، مهیا و تجمیع می‌کند؛

* یک افزونه‌ی PHP (*probe*) که کد PHP را به وسایل اندازه‌گیری مجهز می‌کند؛

برای کار با Blackfire، ابتدا لازم است که `ثبت‌نام کنید <https://blackfire.io/signup>`_.

با اجرای اسکریپت نصب سریع زیر، Blackfire را بر روی رایانه‌ی محلی‌تان نصب کنید:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

این نصاب، ابزار CLI مربوط به Blackfire را بارگیری کرده و PHP probe را بر روی تمام نسخه‌های PHP نصب می‌کند (بدون فعال‌سازی آن).

PHP probe را برای پروژه‌مان فعال کنید:

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

وب سرور را بازراه‌اندازی کنید تا PHP بتواند Blackfire را بار بگیرد:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

ابزار CLI مربوط به Blackfire، نیاز دارد تا با اعتبارنامه‌های **client** شخصی‌تان پیکربندی شود (برای ذخیره‌ی نمایه‌های پروژه‌ی شما در اکانت شخصی‌تان). آن‌ها را در بالای `صفحه‌ی <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`` پیدا کنید و فرمان‌های زیر را با جایگذاری مقادیر مربوطه، اجرا کنید:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    برای دیدن دستورات کامل نصب، `راهنمای نصب مفصل و رسمی <https://blackfire.io/docs/up-and-running/installation>`_ را دنبال کنید. این مستندات هنگامی که می‌خواهید Blackfire را بر روی سرور نصب کنید، به کار می‌آیند.

تنظیم مأمور Blackfire بر روی Docker
--------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

آخرین گام، افزودن سرویس مأمور Blackfire به پشته‌ی Docker Compose است:

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

برای ارتباط با سرور، شما نیاز دارید تا  اعتبارنامه‌های **server** شخصی‌تان را بدست آورید (این اعتبارنامه‌ها، جایی که می‌خواهید نمایه‌ها را در آن ذخیره کنید، مشخص می‌کنند -- می‌توانید برای هر پروژه یکی ایجاد کنید)؛ آن‌ها در پایین `صفحه‌ی <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`` قابل یافتن هستند. آن‌ها را در فایل محلی ``.env.local`` ذخیره کنید:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

حالا می‌توانید کانتینر جدید را راه‌اندازی کنید:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

تعمیر  یک نصبِ خرابِ Blackfire
----------------------------------------------

اگر هنگام profiling خطا دریافت می‌کنید، سطح لاگ را در Blackfire افزایش دهید تا اطلاعات بیشتری در لاگ‌ها دریافت کنید:

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

وب سرور را بازراه‌اندازی کنید:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

و لاگ‌ها را دنبال کنید:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

مجدداً نمایه‌سازی کرده و خروجی لاگ را بررسی کنید.


پیکربندی Blackfire در محیط عمل‌آوری
----------------------------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire به صورت پیشفرض در تمام پروژه‌های SymfonyCloud قرار داده شده است.

اعتبارنامه‌های *server* را به عنوان متغیر محیط تنظیم کنید:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

PHP probe را مثل سایر افزونه‌های PHP فعال کنید:

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

پیکربندی Varnish برای Blackfire
-------------------------------------------

.. index::
    single: SymfonyCloud;Varnish

قبل از اینکه بتوانید مستقر کنید و profiling را شروع کنید، نیاز دارید که از نهان‌سازی HTTP توسط Varnish عبور کنید (آن را دور بزنید). اگر اینکار را نکنید، Blackfire هرگز نمی‌تواند به اپلیکیشن PHP دست یابد. شما تنها درخواست‌های نمایه‌سازی‌ای که از رایانه‌ی محلی‌تان می‌آید را مجاز خواهید شمرد.

IP فعلی‌تان را پیدا کنید:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

و از آن برای پیکربندی Varnish استفاده کنید:

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

حالا می‌توانید استقرار را انجام دهید.

نمایه‌سازی صفحات وب
-------------------------------------

.. index::
    single: Profiling;Web Pages

شما می‌توانید صفحات وب سنتی را از طریق فایرفاکس یا گوگل کروم و از طریق `افزونه‌ی اختصاصی آن‌ها <https://blackfire.io/docs/integrations/browsers/index>`_ ، نمایه‌سازی کنید.

فراموش نکنید که هنگام نمایه‌سازی، بر روی رایانه‌ی محلی‌تان، نهان‌سازی HTTP را در ``public/index.php`` غیرفعال کنید: اگر اینکار را نکنید، شما به جای کدتان، لایه‌ی نهان‌سازی HTTP در سیمفونی را نمایه‌سازی خواهید کرد:

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

برای به دست آوردن یک تصویر بهتر از کارایی اپلیکیشن‌تان در محیط عمل‌آوری، شما باید محیط «عمل‌آوری» را نیز نمایه‌سازی کنید. به صورت پیشفرض، محیط محلی شما از محیط «توسعه» استفاده می‌کند که سربار زیادی ایجاد می‌نماید (غالباً برای جمع‌آوری داده برای نوارابزار اشکال‌زدایی وب و نمایه‌ساز سیمفونی).

.. index::
    single: Symfony CLI;server:prod

تعویض محیط رایانه‌ی محلی‌تان به محیط عمل‌آوری، از طریق تغییر متغیر محیط ``APP_ENV`` در فایل ``.env.local`` قابل انجام است:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

یا می‌توانید از فرمان ``server:prod`` استفاده کنید:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

فراموش نکنید که پس از پایان نمایه‌سازی، به محیط توسعه برگردید:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

نمایه‌سازی منابع API
------------------------------------

.. index::
    single: Profiling;API

بهتر است نمایه‌سازی API یا SPA، با CLI و از طریق ابزار Blackfire CLI که قبلاً نصب کردید، انجام شود:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

فرمان ``blackfire curl`` دقیقاً همان آرگمان‌ها و گزینه‌های `cURL <https://curl.haxx.se/docs/manpage.html>`_  را می‌پذیرد.

مقایسه‌ی کارایی
------------------------------

در گام مربوط به «نهان‌سازی»، ما لایه‌ی نهان‌سازی را برای افزایش کارایی کدمان اضافه کردیم اما تأثیر این تغییر بر کارایی را اندازه‌گیری یا بررسی نکردیم. از آنجایی که همه‌ی ما در حدس‌زدن اینکه چه‌چیزی سریع و چه‌چیزی کند خواهد بود، بسیار بد هستیم، شما ممکن است در نهایت به وضعیتی برسید که برخی بهینه‌سازی‌ها در حقیقت اپلیکیشن‌تان را کندتر کند.

شما باید همیشه تأثیر هر بهینه‌سازی‌ای که انجام می‌دهید را با یک نمایه‌ساز اندازه بگیرید. Blackfire به کمک `ویژگی مقایسه‌اش <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_، به صورت بصری آن را ساده‌تر می‌کند.

نوشتن آزمون‌های کارکردی جعبه‌سیاه
-----------------------------------------------------------------

.. index::
    single: Blackfire;Player

ما نحوه‌ی نوشتن آزمون‌های کارکردی در سیمفونی را دیده‌ایم. Blackfire می‌تواند برای نوشتن سناریوهای مرور‌کردن وب‌سایت استفاده شود که در موقع نیاز می‌توان آن را از طریق `Blackfire player <https://blackfire.io/player>`_ اجرا کرد. بیایید یک سناریو برای ارسال یک کامنت جدید بنویسیم که در محیط توسعه اعتبارسنجی آن از طریق پیوند رایانامه و در محیط عمل‌آوری از طریق مدیر انجام می‌شود.

یک فایل ``.blackfire.yaml`` با محتوای زیر ایجاد کنید:

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

Blackfire player را بارگیری کنید تا قادر به اجرای سناریو به صورت محلی باشید:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

سناریو را در محیط توسعه اجرا کنید:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

یا در محیط عمل‌آوری:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

همچنین سناریوهای Blackfire می‌توانند برای هر درخواست نمایه‌سازی را راه‌اندازی کرده و با افزودن پرچم ``--blackfire``، آزمون‌های کارایی را اجرا کنند.

خودکارسازی بررسی‌های کارایی
-----------------------------------------------------

مدیریت کارایی تنها درباره‌ی بهبود کارایی کد موجود نیست و شامل بررسی اینکه روند کارایی نزولی نباشد نیز هست.

سناریوی نوشته‌شده در بخش قبلی می‌تواند به صورت خودکار در یک جریان‌کار ادغام مستمر (Continuous Integration) یا در محیط عمل‌آوری و بر اساس یک زمان‌بندی منظم اجرا گردد.

همچنین در SymfonyCloud این اجازه داده شده است که وقتی یک شاخه جدید ایجاد می‌کنید یا در محیط عمل‌آوری استقرار انجام می‌دهید، برای بررسی کارایی کد جدید به صورت خودکار  `سناریوها اجرا شوند <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_.

.. sidebar:: بیشتر بدانید

    * `کتاب Blackfire: تشریح کارایی کد PHP <https://blackfire.io/book>`_؛

    * `آموزش تصویری Blackfire در SymfonyCasts <https://symfonycasts.com/screencast/blackfire>`_.
