استخدام RabbitMQ كوسيط الرسائل
=================================================

.. index::
    single: RabbitMQ

RabbitMQ هو وسيط رسائل شائع جدًا يمكنك استخدامه كبديل لـ PostgreSQL.

التحول من PostgreSQL إلى RabbitMQ
--------------------------------------------

لاستخدام RabbitMQ بدلاً من PostgreSQL كوسيط للرسائل:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -6,7 +6,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     options:
                         auto_setup: false
                         use_notify: true

إضافة RabbitMQ إلى Docker Stack
---------------------------------------

.. index::
    single: Docker;RabbitMQ

كما قد تكون خمنت ، نحتاج أيضًا إلى إضافة RabbitMQ إلى مكدس Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -21,3 +21,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

إعادة تشغيل خدمات Docker
---------------------------------------

لإجبار Docker Compose على أخذ حاوية RabbitMQ في الاعتبار ، أوقف الحاويات وأعد تشغيلها:

.. code-block:: terminal

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: terminal
    :class: hide

    $ sleep 10

استكشاف واجهة إدارة الويب RabbitMQ
--------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

إذا كنت تريد رؤية قوائم الانتظار والرسائل التي تتدفق عبر RabbitMQ ، فافتح واجهة إدارة الويب الخاصة به:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

أو من شريط أدوات تصحيح أخطاء الويب:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

استخدم ``guest``/``guest`` لتسجيل الدخول إلى واجهة مستخدم إدارة RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

نشر RabbitMQ
---------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

يمكن إضافة RabbitMQ إلى خوادم الإنتاج عن طريق إضافته إلى قائمة الخدمات:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -18,3 +18,8 @@ files:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

قم بالرجوع إليها في تكوين حاوية الويب أيضًا وقم بتمكين ``amqp`` امتداد PHP :

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - amqp
             - redis
             - blackfire
             - xsl
    @@ -28,6 +29,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

عندما يتم تثبيت خدمة RabbitMQ على مشروع ، يمكنك الوصول إلى واجهة إدارة الويب الخاصة به عن طريق فتح النفق أولاً:

.. code-block:: terminal
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

.. sidebar:: الذهاب أبعد من ذلك

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
