امن‌سازی پشت صحنه‌ی مدیریتی
=====================================================

رابط پشت صحنه‌ی مدیریتی، تنها باید توسط افراد مورد اعتماد قابل دستیابی باشد. امن‌سازی این قسمت از وبسایت، می‌تواند توسط کامپوننت امنیت سیمفونی تأمین گردد.

همچون Twig، کامپوننت امنیت نیز در حال حاضر از طریق وابستگی انتقالی نصب می‌باشد. بیایید آن را به صورت صریح به فایل ``composer.json`` متعلق به پروژه، بیافزاییم:

.. index::
    single: Components;Security
    single: Security

.. code-block:: bash

    $ symfony composer req security

تعریف موجودیت کاربر
------------------------------------

با وجود اینکه شرکت‌کنندگان نمی‌توانند در وب‌سایت برای خود حساب ایجاد کنند، می‌خواهیم یک سیستم احراز هویت کامل را برای مدیریت ایجاد کنیم. بنابراین تنها یک کاربر خواهیم داشت و آن مدیر وب‌سایت است.

اولین گام، تعریف یک موجودیت``User`` است. برای آنکه گیج نشوید، بیاید آن را ``Admin`` بنامیم.

برای ادغام موجودیت ``Admin`` با سیستم احراز هویت امنیت سیمفونی،  این موجودیت باید الزامات خاصی را رعایت کند. برای مثال یک ویژگی ``password`` نیاز دارد.

.. index::
    single: Command;make:user

برای ایجاد موجودیت ``Admin``، به جای فرمان مرسوم ``make:entity`` از فرمان اختصاصی ``make:user`` استفاده کنید:

.. code-block:: bash
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

به سوالات تعاملی پاسخ دهید: ما می‌خواهیم که موجودیت‌های مدیر توسط Doctrine ذخیره شوند (``yes``)، از ``username`` به عنوان نام نمایشی و منحصربه‌فرد مدیران استفاده می‌شود و هر کاربر یک رمزعبور خواهد داشت (``yes``).

کلاس تولیدشده، حاوی متدهایی مانند ``getRoles()``، ``eraseCredentials()`` و تعدادی متد دیگر که برای سیستم احراز هویت سیمفونی لازم هستند، می‌باشد.

اگر می‌خواهید ویژگی‌های بیشتری را به کاربر ``Admin`` اضافه کنید، از فرمان ``make:entity`` استفاده کنید.

بیایید متد ``__toString()`` را اضافه کنیم چرا که EasyAdmin این متد را دوست دارد:

.. code-block:: diff

    --- a/src/Entity/Admin.php
    +++ b/src/Entity/Admin.php
    @@ -75,6 +75,11 @@ class Admin implements UserInterface
             return $this;
         }

    +    public function __toString(): string
    +    {
    +        return $this->username;
    +    }
    +
         /**
          * @see UserInterface
          */

علاوه بر تولید موجودیت ``Admin``، فرمان استفاده‌شده، پیکربندی امنیتی را نیز به‌روزرسانی کرده تا موجودیت به سیستم احراز هویت متصل و سیم‌کشی شود:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 6,7,15,16

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -1,7 +1,15 @@
     security:
    +    encoders:
    +        App\Entity\Admin:
    +            algorithm: auto
    +
         # https://symfony.com/doc/current/security.html#where-do-users-come-from-user-providers
         providers:
    -        in_memory: { memory: null }
    +        # used to reload user from session & other features (e.g. switch_user)
    +        app_user_provider:
    +            entity:
    +                class: App\Entity\Admin
    +                property: username
         firewalls:
             dev:
                 pattern: ^/(_(profiler|wdt)|css|images|js)/

ما اجازه می‌دهیم تا سیمفونی بهترین الگوریتم ممکن برای انکود کردن رمزعبور را انتخاب کند (که در طول زمان تکامل می‌یابد).

زمان تولید یک migration و سپس migrateکردن پایگاه‌داده است:

.. code-block:: bash

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

تولید رمزعبور برای کاربر مدیر
------------------------------------------------------

.. index::
    single: Security;Encoding Passwords

ما یک سیستم اختصاصی برای ایجاد حساب‌های مدیریتی توسعه نمی‌دهیم. دوباره یادآوری می‌کنیم که ما تنها یک مدیر خواهیم داشت. کاربری که وارد می‌شود، ``admin`` خواهد بود و نیاز داریم که رمزعبور را انکود کنیم.

.. index::
    single: Command;security:encode-password

هر چیزی که دوست دارید را به عنوان رمزعبور انتخاب کنید و فرمان زیر را برای تولید رمزعبور انکد‌شده اجرا کنید:

.. code-block:: bash
    :class: answers(admin)

    $ symfony console security:encode-password

.. code-block:: text
    :class: ignore
    :emphasize-lines: 11

    Symfony Password Encoder Utility
    ================================

     Type in your password to be encoded:
     >

     ------------------ ---------------------------------------------------------------------------------------------------
      Key                Value
     ------------------ ---------------------------------------------------------------------------------------------------
      Encoder used       Symfony\Component\Security\Core\Encoder\MigratingPasswordEncoder
      Encoded password   $argon2id$v=19$m=65536,t=4,p=1$BQG+jovPcunctc30xG5PxQ$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s
     ------------------ ---------------------------------------------------------------------------------------------------

     ! [NOTE] Self-salting encoder used: the encoder generated its own built-in salt.


     [OK] Password encoding succeeded

ایجاد یک مدیر
------------------------

.. index::
    single: Symfony CLI;run psql

درج کاربر مدیر از طریق یک بیانیه‌ی SQL:

.. code-block:: bash

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

به escape کردن علامت ``$`` در ستون رمزعبور توجه کنید؛ تمام آن‌ها را escape کنید!

پیکربندی احراز هویت امنیتی
-------------------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

حالا که کاربر مدیر داریم، می‌توانیم پشت صحنه‌ی مدیریتی را امن کنیم. سیمفونی از استراتژی‌های احراز هویت مختلفی پشتیبانی می‌کند. بیایید از روش کلاسیک و محبوب *سیستم احراز هویت فرمی* استفاده کنیم.

فرمان ``make:auth`` را اجرا کنید تا پیکربندی امنیت به‌روزرسانی و قالب ورود به سیستم تولید شده و همچنین یک *احرازکننده‌ی هویت (authenticator)* ایجاد شود:

.. code-block:: bash
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

``1`` را انتخاب کنید تا یک احرازکننده‌ی هویت از نوع فرم ورود به سایت، تولید شود. کلاس احرازکننده‌ی هویت را ``AppAuthenticator`` و کنترلر را ``SecurityController`` بنامید. یک URL به شکل ``/logout`` تولید کنید (``yes``).

این فرمان پیکربندی امنیت را به‌روزرسانی می‌کند تا کلاس‌های تولیدشده سیم‌کشی شوند:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 9

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -16,6 +16,13 @@ security:
                 security: false
             main:
                 anonymous: lazy
    +            guard:
    +                authenticators:
    +                    - App\Security\AppAuthenticator
    +            logout:
    +                path: app_logout
    +                # where to redirect after logout
    +                # target: app_any_route

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#firewalls-authentication

همانطور که در خروجی فرمان اشاره شده است، لازم است که راهِ (route) درون متد ``onAuthenticationSuccess()`` را سفارشی‌سازی کنیم تا هنگامی که کاربر با موفقیت وارد می‌شود، به مسیر مورد نظرمان بازهدایت شود:

.. code-block:: diff

    --- a/src/Security/AppAuthenticator.php
    +++ b/src/Security/AppAuthenticator.php
    @@ -96,8 +96,7 @@ class AppAuthenticator extends AbstractFormLoginAuthenticator implements Passwor
                 return new RedirectResponse($targetPath);
             }

    -        // For example : return new RedirectResponse($this->urlGenerator->generate('some_route'));
    -        throw new \Exception('TODO: provide a valid redirect inside '.__FILE__);
    +        return new RedirectResponse($this->urlGenerator->generate('easyadmin'));
         }

         protected function getLoginUrl()

.. index::
    single: Command;debug:router
    single: Routing;Debug
    single: Debug;Routing

.. tip::

    از کجا بدانم که ``easyadmin``، مسیر مربوط به EasyAdmin است؟ نمی‌دانم. اما من فرمان زیر را که وابستگی و ارتباط میان نامِ راه‌ها (route names) و مسیرها (paths) را نشان می‌دهد، اجرا کردم:

    .. code-block:: bash

        $ symfony console debug:router

افزودن قوانین کنترل مجوز‌‌های دسترسی
----------------------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

یک سیستم امنیتی از دو بخش تشکیل شده است: *احراز هویت (authentication)* و *احراز مجوز (authorization)*. زمانی که کاربر مدیر را ایجاد می‌کردیم، به او نقش ``ROLE_ADMIN`` دادیم. بیایید بخش ``/admin`` را از طریق افزودن قانون به ``access_control``، تنها به کاربرانی که این نقش را دارند محدود کنیم:

.. code-block:: diff
    :emphasize-lines: 8

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -34,5 +34,5 @@ security:
         # Easy way to control access for large sections of your site
         # Note: Only the *first* access control that matches will be used
         access_control:
    -        # - { path: ^/admin, roles: ROLE_ADMIN }
    +        - { path: ^/admin, roles: ROLE_ADMIN }
             # - { path: ^/profile, roles: ROLE_USER }

قوانین ``access_control``، به کمک regular expression‌ها دسترسی را محدود می‌کنند. زمانی که تلاشی برای دسترسی به URLای که با ``/admin`` شروع می‌شود، صورت می‌گیرد، سیستم امنیتی بررسی می‌کند که آیا کاربر واردشده دارای نقش ``ROLE_ADMIN`` هست یا نه.

احراز هویت از طریق فرم ورود به سایت
---------------------------------------------------------------

اگر تلاش کنید که به پشت صحنه‌ی مدیریتی دست پیدا کنید، باید به صفحه‌ی ورود به سایت بازهدایت شوید که در این صفحه از شما خواسته شده است که نام کاربری و رمزعبور را وارد نمایید:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

به کمک ``admin`` و هر رمزعبوری که قبلاً انکود کرده‌اید، وارد شوید. اگر دقیقاً فرمان SQL من را کپی کرده باشید، رمزعبور ``admin`` است.

توجه کنید که باندل EasyAdmin، به صورت خودکار سیستم احراز هویت سیمفونی را تشخیص می‌دهد:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

سعی کنید بر روی پیوند «Sign out» کلیک کنید. شما موفق شدید! یک پشت صحنه‌ی مدیریتی کاملاً امن.

.. index::
    single: Command;make:registration-form

.. note::

    اگر به یک سیستم احراز هویت با قابلیت‌های کامل نیاز دارید، به فرمان ``make:registration-form`` نگاهی بیاندازید.

.. sidebar:: بیشتر بدانید

    * `مستندات امنیت سیمفونی <https://symfony.com/doc/current/security.html>`_؛

    * `آموزش تصویری امنیت در SymfonyCasts <https://symfonycasts.com/screencast/symfony-security>`_؛

    * `چگونه یک فرم ورود به سایت بسازیم <https://symfony.com/doc/current/security/form_login_setup.html>`_ در اپلیکیشن‌های سیمفونی؛

    * `برگه‌تقلب امنیت سیمفونی <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
