إدارة الأداء
=======================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    التحسين المبكر هو أصل كل الشرور.

ربما كنت قد قرأت بالفعل هذا الاقتباس من قبل. لكنني أحب أن أقتبسه بالكامل:

.. epigraph::

    يجب أن ننسى الكفاءات الصغيرة ، قل حوالي 97 ٪ من الوقت: التحسين المبكر هو أصل كل الشرور. ومع ذلك ، يجب ألا نفوت فرصنا في هذه النسبة البالغة 3٪.

    --   Donald Knuth

حتى التحسينات الصغيرة في الأداء يمكن أن تحدث فرقًا ، خاصةً لمواقع التجارة الإلكترونية. الآن وقد أصبح تطبيق سجل الزوار جاهزًا للوقت الأول ، لنرى كيف يمكننا التحقق من أدائه.

أفضل طريقة للعثور على تحسينات الأداء هي استخدام *منشئ ملفات التعريف*. الخيار الأكثر شيوعًا في الوقت الحاضر هو `Blackfire <https://blackfire.io>`_ (*إخلاء تام*: أنا أيضًا مؤسس مشروع Blackfire).

تقديم Blackfire
--------------------

يتكون Blackfire من عدة أجزاء:

* *عميل* يقوم بتشغيل ملفات التعريف (أداة Blackfire CLI أو امتداد متصفح لمتصفح Google Chrome أو Firefox) ؛

* *وكيل* يقوم بإعداد البيانات وتجميعها قبل إرسالها إلى blackfire.io للعرض؛

* امتداد PHP (* probe* ) الذي يصوغ كود PHP.

للعمل مع Blackfire ، تحتاج أولاً إلى `الإشتراك <https://blackfire.io/signup>`_.

قم بتثبيت Blackfire على جهازك المحلي عن طريق تشغيل البرنامج النصي للتثبيت السريع التالي:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

يقوم هذا المثبت بتنزيل أداة Blackfire CLI Tool ثم يقوم بتثبيت مسبار PHP (دون تمكينه) على جميع إصدارات PHP المتوفرة.

تمكين التحقيق PHP لمشروعنا:

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

أعد تشغيل خادم الويب حتى يتمكن PHP من تحميل Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

يلزم تهيئة أداة Blackfire CLI باستخدام بيانات اعتمادك الشخصية **client** (لتخزين ملفات تعريف المشروع الخاصة بك تحت حسابك الشخصي). ابحث عنها أعلى صفحة ``Settings/Credentials`` `page <https://blackfire.io/my/settings/credentials>`_ وقم بتنفيذ الأمر التالي عن طريق استبدال العناصر النائبة:

.. code-block:: terminal
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    للحصول على إرشادات التثبيت الكامل ، اتبع `اتبع دليل التثبيت المفصل الرسمي <https://blackfire.io/docs/up-and-running/installation>`_. تكون مفيدة عند تثبيت Blackfire على الخادم.

إعداد وكيل Blackfire على Docker
-------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

الخطوة الأخيرة هي إضافة خدمة وكيل Blackfire في مكدس Docker Compose:

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

للتواصل مع الخادم ، تحتاج إلى الحصول على بيانات اعتماد ** server** الشخصية (تحدد بيانات الاعتماد هذه المكان الذي تريد تخزين الملفات الشخصية فيه - يمكنك إنشاء واحد لكل مشروع) ؛ يمكن العثور عليها في أسفل صفحة ``Settings/Credentials ``<https://blackfire.io/my/settings/credentials> `_. قم بتخزينها في ملف `` env.local.`` محلي:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

يمكنك الآن تشغيل الحاوية الجديدة:

.. code-block:: terminal
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

إصلاح تثبيت Blackfire غير العامل
---------------------------------------------------

إذا حصلت على خطأ أثناء إنشاء ملفات تعريف ، فقم بزيادة مستوى سجل Blackfire للحصول على مزيد من المعلومات في السجلات:

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

أعد تشغيل خادم الويب:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

وتتبع السجلات:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

الملف الشخصي مرة أخرى وتحقق من إخراج السجل.


تكوين Blackfire في الإنتاج
----------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

يتم تضمين Blackfire افتراضيًا في جميع مشاريع SymfonyCloud.

قم بإعداد بيانات اعتماد * server * كمتغيرات بيئة:

.. code-block:: terminal
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

وقم بتمكين مسبار PHP مثل أي امتداد PHP آخر:

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

إعداد Varnish ل Blackfire
-------------------------------

.. index::
    single: SymfonyCloud;Varnish

قبل أن تتمكن من النشر لبدء التوصيف ، فأنت بحاجة إلى طريقة لتجاوز ذاكرة التخزين المؤقت للورنيش HTTP. إذا لم يكن الأمر كذلك ، فلن تضغط Blackfire أبدًا على تطبيق PHP. ستقوم بتفويض طلبات ملفات التعريف فقط الواردة من جهازك المحلي.

ابحث عن عنوان IP الحالي الخاص بك:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

واستخدامه لتكوين Varnish:

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

يمكنك الآن نشر.

ملفات تعريف صفحات الويب
-------------------------------------------

.. index::
    single: Profiling;Web Pages

يمكنك تعريف صفحات الويب التقليدية من Firefox أو Google Chrome من خلال `الإضافات المخصصة <https://blackfire.io/docs/integrations/browsers/index>`_.

على جهازك المحلي ، لا تنسَ تعطيل ذاكرة التخزين المؤقت لـ HTTP في ``config/packages/framework.yaml`` عند التوصيف: إذا لم يكن الأمر كذلك ، فستقوم بملف تعريف طبقة ذاكرة التخزين المؤقت لـ Symfony HTTP بدلاً من الرمز الخاص بك:

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

للحصول على صورة أفضل لأداء التطبيق الخاص بك في الإنتاج ، يجب عليك أيضًا تعريف بيئة "الإنتاج". تبعًا للإعدادات الافتراضية ، تستخدم البيئة المحلية بيئة "التطوير" ، والتي تضيف حمولة كبيرة (بشكل أساسي لجمع البيانات لشريط أدوات تصحيح الويب وملف التعريف Symfony).

.. index::
    single: Symfony CLI;server:prod

يمكن تحويل جهازك المحلي إلى بيئة الإنتاج عن طريق تغيير متغير البيئة ``APP_ENV`` في ملف ``env.local.``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

أو يمكنك استخدام الأمر ``server:prod``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

لا تنسَ تبديلها مرة أخرى إلى نقطة التطوير عند انتهاء جلسة عمل ملفات التعريف:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

موارد واجهة برمجة التطبيقات
---------------------------------------------------

.. index::
    single: Profiling;API

من الأفضل عمل توصيف API أو SPA على CLI عبر Blackfire CLI Tool التي قمت بتثبيتها مسبقًا:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

يقبل أمر ``blackfire curl`` نفس الحجج والخيارات تمامًا مثل `cURL <https://curl.haxx.se/docs/manpage.html>`_.

مقارنة الأداء
-------------------------

في الخطوة المتعلقة بـ "ذاكرة التخزين المؤقت" ، أضفنا طبقة ذاكرة التخزين المؤقت لتحسين أداء التعليمات البرمجية الخاصة بنا ، لكننا لم نتحقق من تأثير التغيير أو نقيسه. نظرًا لأننا جميعًا سيئون جدًا في تخمين ما سيكون سريعًا وما هو بطيء ، فقد ينتهي بك الأمر في موقف يجعل إجراء بعض التحسينات تطبيقك أبطأ بالفعل.

يجب عليك دائمًا قياس تأثير أي تحسين تقوم به مع المحلل. يجعل Blackfire الأمر أسهل بصريًا بفضل `ميزة المقارنة <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

كتابة الاختبارات الوظيفية للصندوق الأسود
----------------------------------------------------------------------------

.. index::
    single: Blackfire;Player

لقد رأينا كيفية كتابة الاختبارات الوظيفية باستخدام Symfony. يمكن استخدام Blackfire `لكتابة سيناريوهات التصفح التي يمكن تشغيلها عند الطلب عبر `Blackfire player <https://blackfire.io/player>`_. دعنا نكتب سيناريو يقدم تعليقًا جديدًا ويتحقق من صحته عبر رابط البريد الإلكتروني قيد التطوير ، وعبر المسؤول في الإنتاج.

قم بإنشاء ملف `` .blackfire.yaml`` بالمحتوى التالي:

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

قم بتنزيل مشغل Blackfire لتتمكن من تشغيل السيناريو محليًا:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

قم بتشغيل هذا السيناريو في التطوير:

.. code-block:: terminal

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

أو في الإنتاج:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

يمكن لسيناريوهات Blackfire أيضًا تشغيل ملفات تعريف لكل طلب وتشغيل اختبارات الأداء عن طريق إضافة علامة ``blackfire``.

أتمتة التحقيقات الأداء
------------------------------------------

إدارة الأداء لا تتعلق فقط بتحسين أداء الشفرة الحالية ، بل تتعلق أيضًا بالتحقق من عدم ظهور انحدارات في الأداء.

يمكن تشغيل السيناريو المكتوب في القسم السابق تلقائيًا في سير عمل التكامل المستمر أو في الإنتاج بشكل منتظم.

في SymfonyCloud ، يسمح أيضًا بـ `تشغيل السيناريوهات <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ عندما تنشئ فرعًا جديدًا أو تنشر في الإنتاج للتحقق من الأداء من الكود الجديد تلقائيا.

.. sidebar:: الذهاب أبعد من ذلك

    * `كتاب Blackfire: شرح أداء الكود PHP <https://blackfire.io/book>`_؛

    * `البرنامج التعليمي SymfonyCasts Blackfire <https://symfonycasts.com/screencast/blackfire>`_.
