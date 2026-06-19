انشعاب کد
=================

روش‌های زیادی برای سازماندهی گردش‌کار (workflow) تغییرات کد در یک پروژه وجود دارد. اما کار کردن مستقیم بر روی شاخه‌ی اصلیِ Git و استقرار مستقیم در محیط عمل‌آوری بدون آزمودن احتمالاً بهترین روش نیست.

آزمودن تنها مربوط به تست‌های واحد (unit) یا کارکردی (functional) نیست، بلکه شامل بررسی رفتار برنامه با داده‌های محیط عمل‌آوری نیز هست. اگر شما یا سایر `ذی‌نفعان (stakeholders)`_ شما بتوانند اپلیکیشن را دقیقاً همانطور که برای کاربران نهایی مستقر شده است، مشاهده کنند، این موضوع به یک مزیت بزرگ تبدیل شده و به شما امکان استقرار با اطمینان را می‌دهد. این مسئله هنگامی که افراد غیرفنی بتوانند ویژگی‌های جدید را اعتبارسنجی کنند، به شکلی ویژه قدرتمند است.

ما در مراحل بعدی به خاطر سادگی و جلوگیری از تکرار، تمام کارها را در شاخه‌ی اصلی Git ادامه خواهیم داد، اما بیایید ببینیم چگونه این کار می‌تواند بهتر انجام شود.

اتخاذ یک گردش‌کارِ Git
---------------------------------------

یک گردش‌کار ممکن، ایجاد یک شاخه برای هر ویژگی جدید یا رفع اشکال است. این  روش ساده و کارآمد است.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

ایجاد شاخه‌ها
--------------------------

.. index::
    single: Git;branch
    single: Git;checkout

گردش‌کار با ایجاد یک شاخه‌ی Git آغاز می‌شود:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

این فرمان یک شاخه‌ی ``sessions-in-redis`` از روی شاخه ``master`` ایجاد می‌کند. این فرمان کد و پیکربندی زیرساخت را «منشعب (fork)» می‌کند.

ذخیره نشست‌ها (Sessions) در Redis
------------------------------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

همانطور که احتمالاً از نام شاخه حدس زده‌اید، می‌خواهیم محل ذخیره‌ی نشست‌ها را از filesystem به Redis تغییر دهیم.

اقدامات لازم برای واقعیت بخشیدن به این امر، معمول و مورد انتظار است:

#. ایجاد یک شاخه‌ی Git؛

#. در صورت لزوم پیکربندی Symfony را به‌روز کنید؛

#. در صورت لزوم مقداری کد بنویسید و یا کد را به‌روز کنید.

#. پیکربندی PHP را به‌روز کنید (افزونه‌ی Redis PHP را اضافه کنید)؛

#. زیرساخت Docker و SymfonyCloud را به‌روز کنید (سرویس Redis را اضافه کنید)؛

#. آزمودن محلی؛

#. آزمودن از راه دور؛

#. ادغام شاخه به شاخه‌ی اصلی؛

#. استقرار در محیط عمل‌آوری؛

#. شاخه را حذف کنید.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

به شما اجازه می‌دهم تا با مرور وب‌سایت، به صورت محلی آن را بیازمایید. از آنجایی که هیچ تغییر بصری‌ای وجود ندارد و ما هنوز از نشست‌ها استفاده نمی‌کنیم، باید همه چیز باید مانند گذشته کار کند.

استقرار یک شاخه
----------------------------

.. index::
    single: SymfonyCloud;Environment

قبل از استقرار در محیط عمل‌آوری، باید شاخه را در زیرساختی مشابه محیط عمل‌آوری، مورد آزمون قرار دهیم. ما همچنین باید اعتبار اینکه همه چیز برای محیط سیمفونیِ ``prod`` به‌خوبی کار می‌کند را، تأیید کنیم. (وب‌سایت محلی از محیط سیمفونیِ ``dev`` استفاده کرده است).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

حال بیایید یک *محیطِ SymfonyCloud* بر اساس *شاخه‌ی Git* ایجاد کنیم:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

این فرمان یک محیط جدید به صورت زیر ایجاد می‌کند:

* این شاخه‌ی جدید، کد و زیرساخت را از شاخه‌ی فعلی Git به ارث می‌برد (``sessions-in-redis``)؛

* داده‌ها از محیط اصلی (همان محیط عمل‌آوری) و از طریق گرفتن تصویر آنی (snapshot) از کلیه‌ی داده‌های سرویس، از جمله فایل‌ها (مثلاً فایل‌های بارگذاری شده توسط کاربر) و پایگاه‌داده‌ها، بدست می‌آیند؛

* یک خوشه‌ی اختصاصی جدید برای استقرار کد، داده‌ها و زیرساخت ایجاد شده است.

از آنجا که استقرار همان مراحل استقرار به محیط عمل‌آوری را دنبال می‌کند، migrateکردن پایگاه‌داده نیز اجرا می‌شود. این یک روش عالی برای تأیید این است که migration‌ها با داده‌های محیط عمل‌آوری کار می‌کنند.

محیط‌های غیر از محیطِ اصلی (non-``master`)، بسیار شبیه به ``master`` هستند، به جز برخی از تفاوت‌های کوچک: به عنوان مثال، رایانامه‌ها به طور پیشفرض ارسال نمی‌شوند.

.. index::
    single: Symfony CLI;open:remote

زمانیکه استقرار تمام شد، شاخه‌ی جدید را در یک مرورگر باز کنید:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

توجه داشته باشید که تمام فرامین SymfonyCloud، بر روی شاخه‌ی فعلی Git کار می‌کنند. این فرمان، URL مستقرشده برای شاخه‌ی ``sessions-in-redis`` را باز می‌کند؛ این URL شبیه به ``/https://session-in-redis-xxx.eu.s5y.io`` خواهد بود.

وب‌سایت را در این محیط جدید بیازمایید، باید تمام داده‌هایی را که در محیط اصلی (master) ایجاد کرده‌اید، مشاهده کنید.

اگر کنفرانس‌های بیشتری را به محیط ``master`` اضافه کنید، آن‌ها در محیط ``sessions-in-redis`` ظاهر نمی‌شوند و همچنین بالعکس. محیط‌ها مستقل و ایزوله هستند.

اگر کدِ روی شاخه‌ی اصلی تکامل یابد، همیشه می‌توانید شاخه‌ی Git را پایه‌گذاریِ مجدد (rebase) کنید. سپس نسخه‌ی به‌روز‌شده را مستقر کرده و تداخلات (conflicts) را هم برای کد و هم برای زیرساخت، مرتفع کنید.

.. index::
    single: Symfony CLI;env:sync

حتی شما می‌توانید داده‌ها را از شاخه‌ی اصلی، به محیط ``sessions-in-redis`` همگام‌سازی کنید:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

اشکال‌زدایی استقرارهای محصول نهایی، قبل از استقرار
-----------------------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

به طور پیش فرض، تمام محیط‌های SymfonyCloud، از همان تنظیمات محیط ``master``/``prod`` (یا همان محیط سیمفونیِ ``prod``) استفاده می‌کنند. این به شما این امکان را می‌دهد تا برنامه را در شرایط واقعی آزمایش کنید. این موضوع،  احساس توسعه و آزمایش مستقیم روی سرورهای نهایی را به شما القا می‌کند، اما بدون خطرات مرتبط با انجام واقعی این کار. این مرا به یاد روزهای خوب قدیمی می‌اندازد که از طریق FTP این کار را انجام می‌دادیم.

.. index::
    single: Symfony CLI;env:debug

در صورت بروز مشکل ، ممکن است بخواهید به محیط Symfony `` dev`` بروید:

.. code-block:: bash

    $ symfony env:debug

پس از اتمام کارتان، به تنظیمات محیط عمل‌آوری برگردید:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **هرگز** محیط ``dev`` و  نمایه‌ساز سیمفونی را در شاخه ``master`` فعال نکنید؛ این کار برنامه شما را بسیار کند و تعداد زیادی از آسیب‌پذیری‌های امنیتی جدی را ممکن می‌کند.

آزمودن استقرارهای محصول نهایی، قبل از استقرار
------------------------------------------------------------------------------------

دسترسی به نسخه‌ی آینده وب‌سایت با داده‌های عمل‌آوری، فرصت‌های زیادی را ایجاد می‌کند: از آزمون رگرسیون بصری (visual regression testing) گرفته تا آزمون کارایی (performance testin). `Blackfire <https://blackfire.io>`_ یک ابزار عالی برای انجام این کار است.

برای کسب اطلاعات بیشتر در مورد نحوه استفاده از Blackfire برای تست کدهای خود قبل از استقرار، به گام «کارایی» مراجعه کنید.

ادغام در محصول نهایی
-------------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

هنگامی که از تغییرات شاخه راضی شدید، کد و زیرساخت را در شاخه‌ی اصلی Git ادغام کنید:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

و استقرار:

.. code-block:: bash

    $ symfony deploy

هنگام استقرار، تنها تغییرات در کد و زیرساخت به SymfonyCloud ارسال می‌شود؛ داده‌های دیگر به هیچ وجه تحت تأثیر قرار نمی‌گیرند.

تمیزکاری
----------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

در آخر با پاک کردن شاخه‌ی Git و محیط SymfonyCloud تمیزکاری کنید:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: بیشتر بدانید

    * `شاخه‌زنی در Git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_؛

    * `Redis docs <https://redis.io/documentation>`_.

.. _`ذی‌نفعان (stakeholders)`: https://en.wikipedia.org/wiki/Project_stakeholder
