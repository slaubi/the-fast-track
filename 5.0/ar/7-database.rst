إعداد قاعدة بيانات
==================================

.. index::
    single: Database

يدور موقع سجل زوار المؤتمر حول جمع التعليقات أثناء المؤتمرات. نحتاج إلى تخزين التعليقات التي ساهم بها حضور المؤتمر في مخزن دائم.

يتم وصف التعليق بشكل أفضل من خلال بنية بيانات ثابتة: المؤلف ، بريده الإلكتروني ، نص الملاحظات ، وصورة اختيارية. نوع البيانات التي يمكن تخزينها بشكل أفضل في محرك قاعدة بيانات علائقية تقليدي.

PostgreSQL هو محرك قاعدة البيانات الذي سنستعمل.

إضافة PostgreSQL  إلى Docker Compose
--------------------------------------------

.. index::
    single: Docker;PostgreSQL

على الجهاز المحلي لدينا ، قررنا استخدام Docker لإدارة الخدمات. قم بإنشاء ملف ``docker-compose.yaml`` وإضافة PostgreSQL كخدمة:

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

سيؤدي هذا إلى تثبيت خادم PostgreSQL في الإصدار 11 وتكوين بعض متغيرات البيئة التي تتحكم في اسم قاعدة البيانات وبيانات الاعتماد. القيم لا تهم حقا.

نكشف أيضًا منفذ PostgreSQL (`` 5432``) من الحاوية container إلى المضيف المحلي. سيساعدنا ذلك في الوصول إلى قاعدة البيانات من الجهاز الخاص بنا.

.. note::

    يجب أن يكون قد تم تثبيت ملحق `pdo_pgsql`` عندما تم إعداد PHP في خطوة سابقة.

بدء Docker Compose
---------------------

بدء Docker Compose في الخلفية (`` -d``):

.. code-block:: bash

    $ docker-compose up -d

انتظر قليلاً للسماح بتشغيل قاعدة البيانات والتحقق من أن كل شيء يسير على ما يرام:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

إذا لم تكن هناك حاويات قيد التشغيل أو إذا كان عمود ``State`` لا يقرأ ``Up`` ، فتحقق من سجلات Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

الوصول إلى قاعدة البيانات المحلية
--------------------------------------------------------------

قد يكون استخدام الأداة المساعدة لسطر الأوامر ``psql`` مفيدًا من وقت لآخر. ولكن عليك أن تتذكر بيانات الاعتماد واسم قاعدة البيانات. أقل وضوحا ، تحتاج أيضا إلى معرفة المنفذ المحلي الذي تديره قاعدة البيانات على المضيف. يختار Docker منفذًا عشوائيًا حتى تتمكن من العمل على أكثر من مشروع باستخدام  PostgreSQL في نفس الوقت (المنفذ المحلي هو جزء من مخرجات ``docker-compose ps``).

إذا قمت بتشغيل ``psql`` عبر Symfony CLI ، فلن تحتاج إلى تذكر أي شيء.

يقوم Symfony CLI بالكشف تلقائيًا عن خدمات Docker التي يتم تشغيلها للمشروع ويكشف متغيرات البيئة التي يحتاجها ``psql`` للاتصال بقاعدة البيانات.

.. index::
    single: Symfony CLI;run psql

بفضل هذه الاتفاقيات، أصبح الوصول إلى قاعدة البيانات عبر ``symfony run`` أسهل كثيرًا:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

إضافة PostgreSQL إلى SymfonyCloud
-----------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

بالنسبة للبنية التحتية للإنتاج على SymfonyCloud ، يجب إضافة خدمة مثل PostgreSQL في ملف ``.symfony/services.yaml`` الفارغ حاليا:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

خدمة ``db`` هي قاعدة بيانات PostgreSQL في الإصدار 11 (مثل Docker) التي نريد توفيرها في حاوية صغيرة بسعة 1 جيجابايت من القرص.

نحتاج أيضًا إلى "ربط" قاعدة البيانات بحاوية التطبيق الموضحة في ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

تتم الإشارة إلى خدمة ``db`` من النوع ``postgresql`` على أنها ``database`` في حاوية التطبيق.

الخطوة الأخيرة هي إضافة ملحق ``pdo_pgsql`` إلى وقت تشغيل PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

ها هو الفرق الكامل لتغييرات ``.symfony.cloud.yaml``:

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

قم بتنفيذ هذه التغييرات ثم أعد النشر إلى SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

الوصول إلى قاعدة بيانات SymfonyCloud
--------------------------------------------------------

يعمل PostgreSQL الآن محليًا عبر Docker وفي الإنتاج على SymfonyCloud.

كما رأينا للتو ، فإن تشغيل ``symfony run psql`` يتصل تلقائيًا بقاعدة البيانات التي يستضيفها Docker بفضل متغيرات البيئة المتاحة بواسطة ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

إذا كنت ترغب في الاتصال بـ PostgreSQL المستضافة على حاويات الإنتاج ، يمكنك فتح نفق SSH بين الجهاز المحلي والبنية الأساسية لــ SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

بشكل افتراضي ، لا يتم كشف خدمات SymfonyCloud كمتغيرات بيئة على الجهاز المحلي. يجب عليك القيام بذلك بشكل صريح باستخدام علامة ``--expose-env-vars``. لماذا ؟ الاتصال بقاعدة بيانات الإنتاج عملية خطيرة. يمكنك العبث مع بيانات *حقيقية*. الزامية طلب العلامة هو ما يؤكد أن هذا *بالفعل* ما تريد القيام به.

الآن ، قم بالاتصال بقاعدة بيانات PostgreSQL عن بعد من خلال ``symfony run psql`` كما كان من قبل:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

عند الانتهاء ، لا تنس إغلاق النفق:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    لتشغيل بعض استعلامات SQL في قاعدة بيانات الإنتاج بدلاً من الحصول على محث الأوامر ، يمكنك أيضًا استخدام أمر``symfony sql``.

فضح متغيرات البيئة
----------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

يعمل Docker Compose و SymfonyCloud بسلاسة مع Symfony بفضل متغيرات البيئة.

تحقق من كل متغيرات البيئة المكشوفة بواسطة ``symfony`` عن طريق تنفيذ ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

تتم قراءة متغيرات البيئة ``PG*`` بواسطة الأداة المساعدة ``psql``. ماذا عن الآخرين؟

عندما يكون النفق مفتوحًا لـ SymfonyCloud مع مجموعة العلامة ``--expose-env-vars``، فإن الأمر ``var:export`` يُرجع متغيرات البيئة البعيدة:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: الذهاب أبعد من ذلك

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `PostgreSQL وثائق دعم <https://www.postgresql.org/docs/>`_؛

    * `أوامر docker-compose <https://docs.docker.com/compose/reference/>`_.
