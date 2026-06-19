إستكشاف الأخطاء وإصلاحها
==============================================

إعداد المشروع بوجود الأدوات المناسبة لإصلاح المشكلات.

تثبيت المزيد من الاعتمادات
-------------------------------------------------

تذكر بأنه قد تم إنشاء المشروع بعدد قليل من المعتمدات (التبعيات). لا محرك نموذج. لا أدوات إصلاح الأخطاء. لا نظام تسجيل. الفكرة أنه يمكنك إضافة تبعيات عندما تريد. لماذا سوف تعتمد علي محرك نموذج اذا كنت تقوم بإنشاء HTTP API او اداة CLI؟

كيف نضيف المزيد من التبعيات؟ عن طريق مؤلف Composer. بالاضافة الي حزمة كومبوزر Composer "الاساسية" ("regular")، سوف نقوم بالعمل مع نوعين "خاصين" من الحزم:

* *مكونات سيمفوني*: الحزم التي تنفذ الصفات الاساسية وعمليات منخفضة المستوي الي تحتاجها معظم البرامج (routing, console, HTTP client, mailer, cache, ...)

* *رزم سيمفوني*: حزم تقوم بإضافة مميزات عالية المستوي أو توفير التكامل مع المكتبات الخارجية (رزم يتم مشاركتها في الغالب بواسطة المجتمع).

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

في البداية، لنقوم بإضافة محلل سيمفوني، منقذ للوقت عندما تريد معرفة السبب الرئيسي لمشكلة معينة:

.. code-block:: terminal

    $ symfony composer req profiler --dev

``محلل`` هو إسم مستعار يشير الي حزمة `symfony/profiler-pack``.

*الاسماء المستعارة (Aliases)* لا تعتبر خاصية كمبوزر، ولكن الفكرة مقدمة بواسطة سيمفوني لجعل حياتك أسهل. الاسماء المستعارة هي اختصارات للحزم الشائعة في كموزر. تريد ORM للتطبيق الخاص بك؟ إستدعي ``orm``. تريد ان تقوم ببناء API؟ أستدعي ``api``. يتم حل هذه الاسماء المستعارة بشكل تلقائي لواحدة او اكثر من حزم كمبوزر. إنها اختيارات مُقررة بواسطة فريق سيمفوني الاساسي.

ومن المميزات الانيقة الاخري انه يمكنك حزف مورد ``سيمفوني`` (``symfony`` vendor). إستدعي ``cache`` بدلاً من ``symfony/cache``.

.. tip::

    هل تتذكر بأننا قمنا بذكر مكون إضافي في composer يسمي ``symfony/flex``؟ الاسماء المستعارة واحدة من مميزاتها.

فهم بيئات عمل سيمفوني
---------------------------------------

.. index::
    single: Symfony Environments

هل لاحظت علم ``--dev`` في أمر ``composer req``؟ بما أن محلل سيمفوني مفيد فقط أثناء عملية التطوير، نريد أن نتجنب تنصيبه (تسطيبه) في الانتجاية.

يدعم سيمفوني مفهوم *بيئات العمل* (*environments*). بشكل افتراضي، تقوم بدعم ثلاثة، ولكنك يمكنك إضافة العدد الذي تريده: ``dev``، ``prod``، و ``test``. كل البيئات تشارك نفس الرمز البرمجي، ولكنهم يقدمون *إعدادات* (*configurations*) مختلفة.

فعلى سبيل المثال، جميع أدوات تصحيح الأخطاء متاحة في بيئة عمل ``dev``. ولكن في ال ``prod``، تم تحسين التطبيق من أجل الاداء.

التبديل من بيئة عمل الي أخري يمكن ان يتم عن طريق تغير متغير بيئة العمل ``APP_ENV``.

عندما تقوم بالنشر على Upsun، يتم تغير بيئة العمل (المخزنة في ``APP_ENV``) بشكل تلقائي الي ``prod``.

إدارة إعدادات بيئة عمل
-----------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

يمكن أن يتم ضبط متغير ``APP_ENV`` عن طريق استخدام متغيرات بيئة عمل "حقيقية" في شاشة الاومر الخاصة بيك:

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

استخدام متغيرات بيئة عمل حقيقية تعتبر افضل طريقة لتعيين قيم مثل ``APP_ENV`` علي خوادم الانتاجية. ولكن علي اجهزة التطوير، قد يكون الاضطرار لتعين العديد من متغيرات بيئة العمل امراً مرهقاً. وعوضاً عن ذلك، قم بتعريفهم في ملف الـ ``.env``.

بشكل منطقي تم إنشاء ملف الـ ``.env`` لكَ بشكل اوتماتيكي عندم أُنشئ المشروع.

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    يمكن لأي حزمة اضافة العديد من متغيرات بيئة العمل لهذا الملف بفضل وصفتهم التي تُستخدم بواسطة سيمفوني فليكس (Symfony Flex).

يتم الزام ملف الـ ``.env`` للمستودع (repository) ويصف القيم *المبدئية* من الانتاجية. يمكنك تجاوز هذه القيم عن طريق إنشاء ملف ``.env.local``. يجب ألا يتم الزام او رفع هذا الملف لذلك يقوم ملف الـ``.gitignore`` بتجاهله مسبقاً.

لا تخزّن ابداً قيم سرية او حساسة في هذه الملفات. سنري كيفية ادارة الاسرار في خطوة أخري.

تسجيل جميع الأشياء
----------------------------------

.. index::
    single: Logger

خارج الصندوق، تكون إمكانيات التسجيل وتصحيح الاخطاء محدودة علي المشاريع الجديدة. لنقوم بإضافة المزيد من الادوات لتساعدنا علي التحقق من المشاكل اثناء التطوير، ولكن أيضاً في الانتاجية:

.. code-block:: terminal

    $ symfony composer req logger

.. index::
    single: Components;Debug
    single: Debug

ادوات استقصاء وتصحيح المشاكل، لنقوم بتنصيبهم فقط في التطوير:

.. code-block:: terminal

    $ symfony composer req debug --dev

استكشاف ادوات تصحيح الاخطاء الخاصة بسيمفوني
---------------------------------------------------------------------------------

لو قمت بتحديث الصفحة الرئيسية الان، يجب أن تري شريط ادوات في أسفل الشاشة:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

أول شئ من الممكن أن تلاحظه هو الـ **404** بالاحمر. تذكر بأن هذه الصفحة هي عنصر نائب لاننا لم نعرف الصفحة الرئيسية بعد. حتي لو أن الصفحة المبدئية التي تُرحبك تبدو جميلة، تظل صفحة خطأ. لذا فإن رمز حالة الـ HTTP الصحيح هو 404 وليس 200. بفضل شريط ادوات التصحيح، لديك المعلومات علي الفور.

لو قمت بالضغط علي علامة التعجب الصغيرة، سوف تحصل علي رسالة الخطأ "الحقيقية" كجزء من التجسيلات الموجودة في أداة تعريف (محلل) سيمفوني (Symfony profiler). إن كنت تريد ان تري حزمة التعقب، إضغط علي رابط "الاعتراض" (Exception) علي القائمة اليسري.

كلما كان هناك مشكلة في الرمز البرمجي الخاص بك، سوف تري صفحة اعتراض (استثناء) مثل التالية التي تعطيك كل شئ تحتاجة حتي تفهم المشكلة ومن اين تأتي:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

خذ بعض من الوقت لاستكشاف المعلومات الموجودة داخل محلل سيمفوني (Symfony profiler) عن طريق الضغط بالجوار.

.. index::
    single: Symfony CLI;server:log

التجسيلات مفيدة أيضاً في جلسات استقصاء وتصحيح الاخطاء. يمتلك سيمفوني علي أمر مناسب لتعقب (طباعة tail) كل التسجيلات (من الخادم، PHP، وتطبيقك):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

لنقوم بتجربة صغيرة. قم بفتح ``public/index.php`` وعطل او اكسر كود PHP الموجود هناك (علي سبيل المثال قم بإضافة كلمة foobar في الوسط). قم بتحديث الصفحة في المتصفح وراقب مجري التسجيلات:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

النتائج (المُخرجات) ملونة بشكل جميل لجذب انتباهك للاخطاء.

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

هناك مساعد تصحيح آخر وهو دالة سيمفوني ``dump()``. إنها متاحة دائماً وتمكنك من طباعة متغيرات معقدة بشكل جميل وتفاعلي.

بشكل مؤقت غير ``public/index.php`` لطباعة كائن الطلب:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -18,5 +18,8 @@ if ($_SERVER['APP_DEBUG']) {
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);
    +
    +dump($request);
    +
     $response->send();
     $kernel->terminate($request, $response);

عند تحديث الصفحة، تلاحظ ايقونة الـ"target" الجديدة في شريط الادوات؛ تتيح لك فحص التفريغ. إضغط عليها للحصول علي صفحة كاملة حيث يصبح التنقل (التصفح) اسهل:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

استرجع التغييرات قبل تنفيذ التغييرات الاخري التم تم إجراؤها في هذه الخطوة:

.. code-block:: terminal

    $ git checkout public/index.php

إعدادات إطار عملك
--------------------------------

في بيئة التطوير، عندما يحدث استثناء او اعتراض، يقوم سيمفوني بعرض صفحة تحتوي علي رسالة هذا الاعتراض ومسار تتبعه. عند عرض مسار ملف، يقوم بإضافة رابط يقوم بفتح المف عند السطر الصحيح في الـ IDE المفضل لك. للاستفادة من هذه الميزة، قم بإعداد الـ IDE الخاص بك. يدعم سيموفني العديد من الـ IDEs خارج الصندوق؛ فأنا استخدم Visual Studio Code من اجل هذا المشروع:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -6,3 +6,4 @@ max_execution_time=30
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

لا تقتصر الملفات المرتبطة علي الاستثناءات. علي سبيل المثال، المتحكم (وحدة التحكم) في شريط ادوات استقصاء وتصحيح الاخطاء يصبح قابل للضغط بعد اعداد الـIDE.

تصحيح أخطاء الإنتاج
------------------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;logs
    single: Symfony CLI;ssh

إستقصاء وتصحيح اخطاء خوادم الانتاجية أكثر صعوبة دائماً. ليس لديك حق الوصول لمحلل سيموفني علي سبيل المثال. السجلات اقل طولاً (verbose). ولكن طباعة او تذيل (tailing) السجلات ممكن:

.. code-block:: terminal
    :class: ignore

    $ symfony logs

يمكنك حتي الاتصال بواسطة SSH علي حاوية الويب:

.. code-block:: terminal
    :class: ignore

    $ symfony ssh

لا تقلق، فلا يمكنك كسر أي شئ بسهولة. اغلب نظام الملفات للقراءة فقط. لن يمكنك اجراء تصحيح سريع ومهم في الانتاجية. ولكن سوف تتعلم طريقة افضل لاحقا في الكتاب.

.. sidebar:: الذهاب أبعد من ذلك

    * `SymfonyCasts Environments and config files Config Files tutorial <https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files>`_؛

    * `SymfonyCasts Environment Variables tutorial <https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables>`_؛

    * `SymfonyCasts Web Debug Toolbar and Profiler tutorial <https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler>`_؛

    * `Managing multiple .env files <https://symfony.com/doc/current/configuration.html#managing-multiple-env-files>`_ in Symfony applications.
