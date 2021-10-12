تفرع الكود
===================

هناك العديد من الطرق لتنظيم سير عمل تغييرات الكود في المشروع. لكن العمل مباشرة على الفرع الرئيسي (master) في GIT والنشر المباشر للإنتاج (production) دون اختبار ليست ربما أحسن طريقة.

لا يقتصر الاختبار على اختبارات الوحدات أو الاختبارات الوظيفية فحسب ، بل يتعلق أيضًا بالتحقق من سلوك التطبيق مع بيانات الإنتاج (production). إذا تمكنت أنت أو أصحاب المصلحة `stakeholders`_  من تصفح التطبيق تمامًا كما سيتم نشره للمستخدمين النهائيين ، فستصبح هذه ميزة كبيرة ستمكنك من النشر بثقة. انه شيء رائع عندما يتمكن الأشخاص الغير التقنيين من التحقق من صحة الميزات الجديدة.

سنواصل القيام بكل العمل في الفرع الرئيسي (master) ل Git في الخطوات التالية للتبسيط وتجنب التكرار، ولكن دعونا نرى كيف يمكننا تحسين ما قمنا به.

اعتماد سيرة عمل لنظام GIT.
--------------------------------------------

سير عمل بسيط وفعال ممكن استعماله هو إنشاء فرع واحد لكل وظيفة جديدة أو إصلاح خلل.

إنشاء فروع
-------------------

.. index::
    single: Git;branch
    single: Git;checkout

يبدأ سير العمل بإنشاء فرع Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: bash

    $ git checkout -b sessions-in-db

يقوم هذا الأمر بإنشاء فرع " sessions-in-redis ''  من الفرع `` الرئيسي / master  ''. فإنه ينشئ فرع "fork" للكود وتكوين (configuration) البنية التحتية.

جلسات التخزين في قاعدة البيانات
----------------------------------------------------------

.. index::
    single: Session;Database

كما كنت قد خمنت إنطلاقا من إسم الفرع، نريد تغيير تخزين الجلسات من نظام الملفات إلى قاعدة البيانات (نستخدم PostgreSQL هنا).

الخطوات اللازمة لتحقيق المبتغى هي نموذجية:

#. إنشاء فرع في GIT؛

#. تحديث تكوين Symfony إذا إستلزم الأمر؛

#. كتابة و / أو تحديث بعض الكود إذا إستلزم الأمر؛

#. تحديث تكوين PHP إذا احتجنا (إضافة إمتداد PostgreSQL)؛

#. تحديث البنية التحتية في Docker و SymfonyCloud إذا إحتجنا (إضافة خدمة PostgreSQL)؛

#. الإختبار محليًا

#. الإختبار عن بعد؛

#. دمج الفرع في الفرع الرئيسي (Master)؛

#. النشر في الإنتاج (production)؛

#. حذف الفرع؛

لتخزين الجلسات في قاعدة البيانات ، قم بتغيير تكوين ``session.handler_id`` للإشارة إلى قاعدة البيانات DSN:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

لتخزين الجلسات في قاعدة البيانات ، نحتاج إلى إنشاء جدول ``sessions``. افعل ذلك من خلال ترحيل Doctrine:

.. code-block:: bash

    $ symfony console make:migration

قم بتحرير الملف لإضافة إنشاء الجدول في طريقة ``up ()``:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,14 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs

    +        $this->addSql('
    +            CREATE TABLE sessions (
    +                sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    +                sess_data BYTEA NOT NULL,
    +                sess_lifetime INTEGER NOT NULL,
    +                sess_time INTEGER NOT NULL
    +            )
    +        ');
         }

         public function down(Schema $schema): void

ترحيل قاعدة البيانات:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

اختبر محليًا من خلال تصفح الموقع. نظرًا لعدم وجود تغييرات ولأننا لا نستخدم الجلسات حتى الآن ، فكل شيء يسير كما كان من قبل.

.. note::

    لا نحتاج إلى الخطوات من 3 إلى 5 هنا لأننا نعيد استخدام قاعدة البيانات كمخزن للجلسة ، لكن الفصل الخاص باستخدام Redis يوضح مدى سهولة إضافة واختبار ونشر خدمة جديدة في كل من Docker و SymfonyCloud.

نظرًا لأن الجدول الجديد لا تتم "إدارته" بواسطة Doctrine ، فيجب علينا تكوين Doctrine لعدم إزالته في ترحيل قاعدة البيانات التالية:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/doctrine.yaml
    +++ b/config/packages/doctrine.yaml
    @@ -5,6 +5,8 @@ doctrine:
             # IMPORTANT: You MUST configure your server version,
             # either here or in the DATABASE_URL env var (see .env file)
             #server_version: '13'
    +
    +        schema_filter: ~^(?!session)~
         orm:
             auto_generate_proxy_classes: true
             naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware

التزم بتغييراتك في الفرع الجديد:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

نشر فرع
-------------

.. index::
    single: SymfonyCloud;Environment

قبل النشر في الإنتاج ، يجب أن نختبر الفرع على نفس البنية التحتية المطابقة للإنتاج. يجب أن نتحقق أيضًا أن كل شيء يعمل بشكل جيد لبيئة prod ل Symfony  (الموقع المحلي يستخدم بيئة dev ل Symfony.

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

الآن ، دعنا ننشئ * بيئة ل SymfonyCloud * على أساس * فرع Git * :

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-db --no-interaction

.. code-block:: bash

    $ symfony env:create

هذا الأمر ينشئ بيئة جديدة على النحو التالي:

* يرث هذا الفرع الكود والبنية التحتية من فرع Git الحالي"sessions-in-db"؛

* تأتي البيانات من الفرع الرئيسي (master) (المعروفة أيضًا بالإنتاج - production ) من خلال أخذ لقطة متسقة لجميع بيانات الخدمة ، بما في ذلك الملفات ( على سبيل المثال الملفات التي يقوم المستخدم بتحميلها) وقواعد البيانات؛

* يتم إنشاء مجموعة جديدة (cluster) مخصصة لنشرالكود والبيانات والبنية التحتية.

نظرًا لأن النشر يتبع نفس خطوات النشر في الإنتاج ، فسيتم تنفيذ عمليات ترحيل قاعدة البيانات أيضًا. هذه طريقة رائعة للتحقق من أن عمليات الترحيل تعمل مع بيانات الإنتاج.

الفروع الغير `` الرئيسية '' تشبه إلى حد كبير `` الرئيسية '' باستثناء بعض الاختلافات الصغيرة: على سبيل المثال ، لا يتم إرسال رسائل البريد الإلكتروني افتراضيًا.

.. index::
    single: Symfony CLI;open:remote

بمجرد الإنتهاء من النشر ، افتح الفرع الجديد في متصفح:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

لاحظ أن جميع أوامر SymfonyCloud تعمل على فرع Git الحالي. يفتح هذا الأمر عنوان URL المنشور لفرع  "sessions-in-db"؛ سيبدو عنوان URL كما يلي: "https://sessions-in-db-xxx.eu.s5y.io/"

اختبر الموقع على هذه البيئة الجديدة ، يجب أن تشاهد جميع البيانات التي أنشأتها في البيئة الرئيسية.

إذا قمت بإضافة المزيد من المؤتمرات حول البيئة "الرئيسية / master  '' ، فلن تظهر في بيئة "sessions-in-db" والعكس صحيح. البيئات مستقلة ومعزولة.

إذا تطور الكود في  الفرع الرئيسي (Master) ، يمكنك دائمًا إعادة بناء (Rebase) فرع Git ونشر الإصدار المحدث ، وحل كل التعارضات (conflicts) لكل من الكود والبنية التحتية.

.. index::
    single: Symfony CLI;env:sync

يمكنك حتى مزامنة البيانات من الفرع الرئيسي(Master) مرة أخرى إلى بيئة  "sessions-in-db":

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

تصحيح أخطاء عمليات النشر للإنتاج قبل النشر
------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

بشكل افتراضي ، تستخدم جميع بيئات SymfonyCloud نفس الإعدادات مثل بيئة "Master" / "prod   (تعرف أيضًا ببيئة Symfony" prod ''). هذا يسمح لك باختبار التطبيق في ظروف الحياة الحقيقية. يمنحك الشعور بالتطوير والاختبار مباشرة في الإنتاج (production servers) ، ولكن دون المخاطر المرتبطة بها. هذا يذكرني بالأيام الخوالي عندما كنا ننشر عبر FTP.

.. index::
    single: Symfony CLI;env:debug

في حالة وجود مشكلة ، قد ترغب في التغيير  إلى بيئة "dev" ل Symfony :

.. code-block:: bash

    $ symfony env:debug

عند الانتهاء ، ارجع إلى إعدادات الإنتاج:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    لا تقوم *ابداً* بتمكين بيئة التطوير (``dev``) ولا تقوم ايضاً بتمكين أداة تعريف سيمفوني علي الفرع الرئيسي (``master``)؛ فهذا سوف يجعل تطبيقك بطئ جداً وسوف يفتح الكثير من الثغرات الأمنية الخطيرة.

اختبار عمليات نشر الإنتاج (production deployments) قبل النشر
------------------------------------------------------------------------------------------

الوصول إلى الإصدار القادم من الموقع مع بيانات الإنتاج (production)  يتيح الكثير من الفرص: من اختبار الانحدار البصري  (visual regression) إلى اختبار الأداء (performance). `Blackfire <https://blackfire.io>`_ هي الأداة المثالية لهذه المهمة.

ارجع إلى خطوة "الأداء (performance)" لمعرفة المزيد حول كيفية استخدام Blackfire لاختبار الكود قبل النشر.

الإدماج في الإنتاج (production)
-----------------------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

عندما تكون راضيًا عن تغييرات في الفرع ، قم بدمج الكود والبنية التحتية مرة أخرى في فرع Git الرئيسي (master):

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-db

وقم بالنشر:

.. code-block:: bash

    $ symfony deploy

عند النشر ، يتم فقط دفع التغييرات فيالكود والبنية الأساسية إلى SymfonyCloud ؛ لا تتأثر البيانات بأي شكل من الأشكال.

التنظيف
--------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

وأخيرًا ، قم بالتنظيف بإزالة فرع Git وبيئة SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-db
    $ symfony env:delete --env=sessions-in-db --no-interaction

.. sidebar:: الذهاب أبعد من ذلك

    * `التفرع في GIT <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_؛

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
