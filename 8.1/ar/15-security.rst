تأمين الواجهة الخلفية للمدير
=====================================================

يجب أن تكون الواجهة الخلفية للمدير قابلة للوصول فقط من قبل الأشخاص الموثوق بهم. يمكن تأمين هذا الجزء من الموقع باستخدام Symfony SecurityComponent.

مثل Twig ، تم تثبيت  ال Security Component  بالفعل عبر تبعيات متعدية (transitive dependencies). دعنا نضيفها إلى ملف `` composer.json '' للمشروع:

.. index::
    single: Components;Security
    single: Security

.. code-block:: terminal

    $ symfony composer req security

تحديد كيان (Entity) المستخدم
---------------------------------------------

حتى إذا لم يتمكن الحاضرون من إنشاء حساباتهم الخاصة على الموقع، فسننشئ نظام مصادقة كامل الوظائف للمدير. لذلك سيكون لدينا مستخدم واحد فقط ، مدير الموقع.

الخطوة الأولى هي تحديد كيان `` المستخدم ''. لتجنب أي ارتباك ، دعنا نسميها `` Admin '' بدلاً من ذلك.

لدمج كيان "المدير / Admin'' مع نظام مصادقة لSymfony Security ، نحتاج إلى اتباع بعض المتطلبات. على سبيل المثال ، يحتاج إلى خاصية "كلمة المرور/ password''.

.. index::
    single: Command;make:user

استخدم الأمر المخصص "make: user '' لإنشاء كيان  Entity "المدير/Admin '' بدلاً من الكيان التقليدي "make: entity'':

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

أجب عن الأسئلة التفاعلية: نريد استخدام Doctrine لتخزين المديرين (" نعم / yes  '') ، استخدم "اسم المستخدم / username ' كإسم فريد للمديرين، وسيكون لكل مستخدم كلمة مرور / password  (" نعم / yes '') .

يحتوي الفصل الذي تم إنشاؤه على طرق مثل " getRoles() "  و '' eraseCredentials() `` ، وبعض الأنواع الأخرى التي يحتاجها نظام مصادقة Symfony.

إذا كنت ترغب في إضافة المزيد من الخصائص إلى المستخدم " Admin '' ، فاستخدم " make:entity ''.

دعنا نضيف طريقة `` __toString() '' كما يطلبها EasyAdmin:

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

بالإضافة إلى إنشاء كيان " المدير '' ، قام الأمر أيضًا بتحديث إعداد الأمان (The security configuration) لتوصيل الكيان بنظام المصادقة:

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

نسمح لـ Symfony باختيار أفضل خوارزمية ممكنة لتشفير كلمات المرور passwords (التي ستتطور بمرور الوقت).

حان الوقت لإنشاء ترحيل (migration) وترحيل قاعدة البيانات:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

إنشاء كلمة مرور password للمستخدم المدير Admin
-------------------------------------------------------------------------

.. index::
    single: Security;Encoding Passwords

لن نطور نظامًا مخصصًا لإنشاء حسابات المدير. مرة أخرى ، سيكون لدينا مدير واحد فقط. سيكون تسجيل الدخول هو " admin '' ونحن بحاجة إلى ترميز encode كلمة المرور password .

.. index::
    single: Command;security:encode-password

اختر ما تريده ككلمة مرور وقم بتشغيل الأمر التالي لإنشاء كلمة المرور المشفرة encoded password:

.. code-block:: terminal
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

إنشاء مدير
-------------------

.. index::
    single: Symfony CLI;run psql

أدخل المستخدم المدير عبر عبارة SQL:

.. code-block:: terminal

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

لاحظ التخليص escaping من علامة " $ '' في عمود كلمة المرور ؛ قم بهذا (escaping) لكل منهم !

إعداد مصادقة الأمان Security Authentication
------------------------------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

الآن بعد أن أصبح لدينا مستخدم مدير ، يمكننا تأمين الواجهة الخلفية للمدير. يدعم Symfony العديد من استراتيجيات المصادقة. دعونا نستخدم نظام مصادقة نموذجي وشائع.

قم بتشغيل الأمر " make: auth '' لتحديث إعداد الأمان (The security configuration ) ، وإنشاء قالب تسجيل دخول ، وإنشاء *authenticator*:

.. code-block:: terminal
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

إختر " 1 '' لإنشاء مصدق نموذج تسجيل الدخول ، واسم ال Authenticator class  " AppAuthenticator '' ، ووحدة التحكم Controller  " SecurityController '' ، وقم بإنشاء عنوان URL " / logout '' (" نعم '').

قام الأمر بتحديث إعداد الأمان The security configuration  لتوصيل الفئات classes  التي تم إنشاؤها:

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

كما ظهر بها إخراج الأمر ، نحتاج إلى تخصيص المسار في طريقة `` onAuthenticationSuccess() `` لإعادة توجيه المستخدم عند تسجيل الدخول بنجاح:

.. code-block:: diff

    --- a/src/Security/AppAuthenticator.php
    +++ b/src/Security/AppAuthenticator.php
    @@ -95,8 +95,7 @@ class AppAuthenticator extends AbstractFormLoginAuthenticator implements Passwor
                 return new RedirectResponse($targetPath);
             }

    -        // For example : return new RedirectResponse($this->urlGenerator->generate('some_route'));
    -        throw new \Exception('TODO: provide a valid redirect inside '.__FILE__);
    +        return new RedirectResponse($this->urlGenerator->generate('admin'));
         }

         protected function getLoginUrl()

.. index::
    single: Command;debug:router
    single: Routing;Debug
    single: Debug;Routing

.. tip::

    كيف أتذكر أن مسار EasyAdmin هو ``admin`` (كما هو محدد في ``App\Controller\Admin\DashboardController``)؟ لا أتذكر. يمكنك النظر للملف وأيضا إذا قمت بتشغيل الأمر التالي الذي يوضح الارتباط بين أسماء المسارات route names  والمسارات paths :

    .. code-block:: terminal

        $ symfony console debug:router

إضافة قواعد التحكم في الوصول إلى التفويض Authorization Access Control Rules
-------------------------------------------------------------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

يتكون نظام الأمان من جزأين: *المُصادقة* و *التفويض*. عند إنشاء مستخدم المدير، أعطينا لهم دور ``ROLE_ADMIN``. لنقصر قسم ``/admin`` على المستخدمين الذين لديهم هذا الدور عن طريق إضافة قاعدة إلى ``access_control``:

.. code-block:: diff
    :emphasize-lines: 8

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -35,5 +35,5 @@ security:
         # Easy way to control access for large sections of your site
         # Note: Only the *first* access control that matches will be used
         access_control:
    -        # - { path: ^/admin, roles: ROLE_ADMIN }
    +        - { path: ^/admin, roles: ROLE_ADMIN }
             # - { path: ^/profile, roles: ROLE_USER }

تُقيد قواعد ``access_control`` إمكانية الوصول عن طريق التعابير النمطية (regular expressions). عند محاولة الاتصال عن طريق رابط يبدأ بـ ``/admin``، سوف يقوم النظام الأمني بالتحقق من دور الـ ``ROLE_ADMIN`` علي المستخدم الذي قام بتسجيل الدخول.

المُصادقة عبر نموذج تسجيل الدخول
------------------------------------------------------------

إذا حاولت الوصول الي خلفية الادارة (admin backend)، يجب ان يتم تحويلك الآن الي صفحة تسجيل الدخول وسيتم مطالبتك بإدخال بينات تسجيل الدخول وكلمة مرور:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

قم بتسجيل الدخول بإستخدام ``admin`` واي كلمة مرور قمت بتشفيرها مُسبقاً. لو قمت بنسخ امر الـ SQL الخاص بي كما هو فكلمة المرور هي ``admin``.

لاحظ أن EasyAdmin تُدرك نظام تصادق سيمفوني بشكل تلقائي:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

حاول أن تضغط علي رابط تسجيل الخروج "Sign out". إنه بحوزتك! خلفية مسؤول (backend admin) مؤمنة بالكامل.

.. index::
    single: Command;make:registration-form

.. note::

    إذا كنت تريد إنشاء نموذج مُصادقة مكتمل المميزات، لُتلقي نظرة علي أمر ``make:registration-form``.

.. sidebar:: الذهاب أبعد من ذلك

    * `مراجع أمان سيمفوني <https://symfony.com/doc/current/security.html>`_؛

    * `SymfonyCasts Security tutorial <https://symfonycasts.com/screencast/symfony-security>`_؛

    * `كيف تنشئ نموذج تسجيل دخول <https://symfony.com/doc/current/security/form_login_setup.html>`_ في تطبيقات سيمفوني؛

    * `ورقة الغش لأمان سيمفوني <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
