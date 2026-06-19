راه‌اندازی یک پایگاه‌داده
==================================================

.. index::
    single: Database

وبسایت کنفرانس Guestbook، در حال گرفتن بازخورد‌ها در طول کنفرانس است. ما نیاز داریم کامنت‌هایی که توسط شرکت‌کنندگان کنفرانس اظهار شده است را در یک انبار دائمی ذخیره‌سازی کنیم.

یک کامنت به بهترین وجه توسط یک داده‌ساختار ثابت بیان می‌شود: یک نویسنده، رایانامه‌ی نویسنده، متن بازخورد و یک تصویر اختیاری. نوعی از داده که می‌تواند به بهترین وجه در یک موتور پایگاه‌داده‌ی رابطه‌ای سنتی ذخیره شود.

PostgreSQL موتور پایگاه‌داده‌ای است که ما از آن استفاده خواهیم کرد.

افزودن PostgreSQL به Docker Compose
-------------------------------------------

.. index::
    single: Docker;PostgreSQL

ما تصمیم گرفتیم که در رایانه‌ی محلی‌مان، از Docker برای مدیریت سرویس‌ها استفاده کنیم. یک فایل ``docker-compose.yaml`` ایجاد کنید و PostgreSQL را به عنوان سرویس به آن بیافزایید:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

این یک سرور PostgreSQL با نسخه‌ی ۱۱ را نصب خواهد کرد و همچنین تعدادی متغیر محیط را برای کنترل نام پایگاه‌داده و اعتبارنامه‌ها (credentials)، پیکربندی می‌کند. مقادیر آن واقعاً اهمیتی ندارند.

همچنین درگاه (port) مربوط به کانتینر PostgreSQL (یعنی درگاه ``5432``) را در اختیار میزبان محلی می‌گذاریم. این کمک می‌کند تا از طریق رایانه‌ی محلی به پایگاه‌داده دسترسی داشته باشیم.

.. note::

    هنگامی که PHP در گام قبلی تنظیم شد، افزونه‌ی ``pdo_pgsql`` می‌بایست نصب می‌گردید.

اجرای Docker Compose
-------------------------

Docker Compose را در پس زمینه (``-d``) اجرا کنید:

.. code-block:: bash

    $ docker-compose up -d

مقداری صبر کنید و اجازه دهید تا پایگاه‌داده راه‌اندازی شود. سپس بررسی کنید که همه چیز به صورت صحیح در حال اجرا است:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

اگر هیچ کانتینری در حال اجرا نیست یا ستون ``State`` دارای مقدار ``Up`` نیست، لاگ‌های Docker را بررسی کنید:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

دسترسی به پایگاه‌داده‌ی محلی
-------------------------------------------------------

گاهی اوقات استفاده از ابزار خط فرمان ``psql``، می‌تواند مفید واقع شود. اما شما باید اعتبارنامه‌ها و نام پایگاه‌داده را به خاطر بیاورید. همچنین باید درگاه محلی پایگاه‌داده‌ی در حال اجرا بر روی میزبان را بدانید. Docker یک درگاه تصادفی را انتخاب می‌کند تا بتوانید به صورت همزمان بر روی بیش از یک پروژه که از PostgreSQL استفاده می‌کند، کار کنید (درگاه محلی بخشی از خروجی فرمان ``docker-compose ps`` است).

اگر فرمان ``psql`` را از طریق رابط خط فرمان سیمفونی اجرا کنید، نیاز نیست که هیچ چیزی را به خاطر آورید.

رابط خط فرمان سیمفونی، به صورت خودکار تمام سرویس‌هایی که پروژه را اجرا می‌کنند، شناسایی کرده و متغیر‌های محیط لازم برای فرمان ``psql`` جهت اتصال به پایگاه‌داده را در اختیار فرمان قرار می‌دهد.

.. index::
    single: Symfony CLI;run psql

به کمک این قراردادها، دسترسی به پایگاه‌داده از طریق ``symfony run`` بسیار راحت‌تر است:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

افزودن PostgreSQL به SymfonyCloud
-----------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

برای زیرساخت عمل‌آوری بر روی SymfonyCloud، افزودن یک سرویس همچون PostgreSQL، باید از طریق فایل ``.symfony/services.yaml`` که در حال حاضر خالی است، انجام شود:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

سرویس ``db`` یک پایگاه‌داده‌ی PostgreSQL با نسخه‌ی ۱۱ (همچون Docker) است که می‌خواهیم بر روی کانتینری کوچک با 1GB دیسک مهیا شود.

همچنین نیاز داریم تا پایگاه‌داده را به کانتینر اپلیکیشن «متصل (link)» کنیم، که این موضوع در فایل ``.symfony.cloud.yaml`` توصیف شده است:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

سرویس ``db`` از نوع ``postgresql`` در کانتینر اپلیکیشن به عنوان ``database`` ارجاع داده شده است.

آخرین گام، اضافه کردن افزونه‌ی ``pdo_pgsql`` به PHP است:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

این diff کاملِ مربوط به تغییرات فایل ``.symfony.cloud.yaml`` است:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

این تغییرات را commit کنید و مجدداً اپلیکیشن را در SymfonyCloud مستقر کنید:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

دسترسی به پایگاه‌داده‌ی SymfonyCloud
-----------------------------------------------------------

حالا PostgreSQL هم بر روی رایانه‌ی محلی‌تان از طریق Docker و هم در محیط عمل‌آوری بر روی SymfonyCloud در حال اجرا است.

همانطور که مشاهده کردیم، به لطف متغیر‌های محیط ارائه‌شده توسط ``symfony run``، اجرای ``symfony run psql`` به صورت خودکار به پایگاه‌داده‌ی میزبانی‌شده توسط Docker متصل می‌شود.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

اگر می‌خواهید به PostgreSQL میزبانی‌شده در کانتینرهای عمل‌آوری متصل شوید، می‌توانید یک تونل SSH میان رایانه‌ی محلی و زیرساخت SymfonyCloud برقرار کنید:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

به صورت پیشفرض، سرویس‌های SymfonyCloud به صورت متغیر‌های محیط در اختیار رایانه‌ی محلی‌تان نیستند. اگر خلاف این را می‌خواهید، باید صریحاً از پرچم ``--expose-env-vars`` استفاده کنید. چرا؟ اتصال به پایگاه‌داده‌ی عمل‌آوری، عملیاتی خطرناک است. شما ممکن است با داده‌های *واقعی* خرابکاری کنید. الزام استفاده از پرچم روشی است که شما از طریق آن، تأیید می‌کنید که *این* همان چیزی است که می‌خواهید.

حالا از طریق فرمان ``symfony run psql``، همچون گذشته به پایگاه‌داده‌ی PostgreSQL ریموت متصل شوید:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

فراموش نکنید که پس از پایان کار، تونل را ببندید:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    برای اجرای تعدادی پرس‌وجوی SQL بر روی پایگاه‌داده‌ی عمل‌آوری، به جای استفاده از شِل (shell)، می‌توانید از فرمان ``symfony sql`` نیز استفاده کنید.

ارائه‌ی متغیر‌های محیط
--------------------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

به لطف متغیر‌های محیط، Docker Compose و SymfonyCloud به صورت یکپارچه با یکدیگر کار می‌کنند.

با اجرای فرمان ``symfony var:export`` تمام متغیر‌های محیط ارائه‌شده توسط ``symfony`` را بررسی کنید:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

متغیر‌های محیط ``PG*`` توسط ابزار ``psql`` خوانده می‌شوند. سایر موارد چطور؟

زمانی که یک تونل به SymfonyCloud همراه با پرچم ``--expose-env-vars`` باز است، فرمان ``var:export`` متغیر‌های محیط ریموت را بازمی‌گرداند:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: بیشتر بدانید

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `مستندات PostgreSQL <https://www.postgresql.org/docs/>`_؛

    * `فرمان‌های docker-compose <https://docs.docker.com/compose/reference/>`_.
