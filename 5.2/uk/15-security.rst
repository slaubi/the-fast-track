Захист панелі керування
============================================

Внутрішній інтерфейс панелі керування має бути доступний тільки довіреним особам. Захист цієї області веб-сайту може бути виконана за допомогою компонента Symfony Security.

Як і для Twig, компонент безпеки вже встановлено через транзитивні залежності. Проте, додаймо його явно, у файлі ``composer.json``:

.. index::
    single: Components;Security
    single: Security

.. code-block:: bash

    $ symfony composer req security

Визначення сутності користувача
------------------------------------------------------------

Попри те що учасники не зможуть створити свої власні облікові записи на веб-сайті, ми створимо повнофункціональну систему аутентифікації для адміністратора. Тому у нас буде тільки один користувач — адміністратор веб-сайту.

Першим кроком є визначення сутності ``User``. Щоб уникнути можливої плутанини, назвімо її ``Admin``.

Щоб інтегрувати сутність ``Admin`` із системою аутентифікації Symfony Security, вона має відповідати певним вимогам. Наприклад, має бути властивість ``password``.

.. index::
    single: Command;make:user

Використовуйте спеціальну команду ``make:user``, щоб створити сутність ``Admin``, замість традиційної ``make:entity``:

.. code-block:: bash
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Дайте відповідь на інтерактивні запитання: ми хочемо використовувати Doctrine для зберігання адміністраторів (``yes``), використовувати властивість ``username`` для відображення унікального імені адміністратору, та чи кожен користувач матиме пароль (``yes``).

Згенерований клас містить такі методи, як ``getRoles()``, ``eraseCredentials()`` і деякі інші, які необхідні для системи аутентифікації Symfony.

Якщо ви хочете додати додаткові властивості до користувача ``Admin`` — використовуйте команду ``make:entity``.

Додаймо метод ``__toString()``, як цього потребує EasyAdmin:

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

На додаток до генерування сутності ``Admin``, команда також оновила конфігурацію безпеки, щоб зв'язати сутність із системою аутентифікації:

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

Ми дозволимо Symfony вибрати найкращий з можливих алгоритмів кодування паролів (який буде розвиватися з часом).

Настав час згенерувати міграцію та застосувати її до бази даних:

.. code-block:: bash

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Генерування пароля для адміністратора
-----------------------------------------------------------------------

.. index::
    single: Security;Encoding Passwords

Ми не будемо розробляти окрему систему для створення облікових записів адміністраторів. Знову ж таки, у нас буде тільки один адміністратор. Логіном буде ``admin``, і нам потрібно закодувати пароль.

.. index::
    single: Command;security:encode-password

Виберіть будь-який пароль, і виконайте наступну команду для генерування закодованого значення:

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

Створення адміністратора
-----------------------------------------------

.. index::
    single: Symfony CLI;run psql

Додайте користувача-адміністратора за допомогою SQL-виразу:

.. code-block:: bash

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Зверніть увагу на екранування знака ``$`` в значенні стовпця password; Екрануйте їх всі!

Налаштування аутентифікації
-----------------------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Тепер, коли у нас є адміністратор, ми можемо захистити панель керування. Symfony підтримує декілька стратегій аутентифікації. Використовуймо класичну й популярну *систему аутентифікації за допомогою форми*.

Виконайте команду ``make:auth``, щоб оновити конфігурацію безпеки, згенерувати шаблон форми входу та *аутентифікатор*:

.. code-block:: bash
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Виберіть ``1``, щоб згенерувати аутентифікатор форми входу, назвіть клас аутентифікатора — ``AppAuthenticator``, контролер — ``SecurityController`` і згенеруйте URL-адресу ``/logout`` (``yes``).

Команда оновила конфігурацію безпеки для налаштування згенерованих класів:

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

Завдяки підказці, що з'явилася після виконання команди, можна зрозуміти, що нам потрібно налаштувати маршрут у методі ``onAuthenticationSuccess()``, щоб перенаправити користувача після успішного входу в систему:

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

    Як я пам'ятаю, що маршрутом EasyAdmin є ``admin`` (як налаштовано в ``App\Controller\Admin\DashboardController``)? Я не пам'ятаю. Ви можете поглянути на файл, але також можна виконати наступну команду, яка показує зв'язок між іменами маршрутів і шляхами:

    .. code-block:: bash

        $ symfony console debug:router

Додавання правил контролю доступу до авторизації
-------------------------------------------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Система безпеки складається з двох частин: *аутентифікація* та *авторизація*. При створенні адміністратора ми дали йому роль ``ROLE_ADMIN``. Дозвольмо доступ до розділу ``/admin`` тільки тим користувачам, які мають цю роль, додавши правило в ``access_control``:

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

Правила ``access_control`` обмежують доступ за допомогою регулярних виразів. При спробі переходу за URL-адресою, що починається з ``/admin``, система безпеки перевірить наявність ролі ``ROLE_ADMIN`` у користувача, що здійснив вхід.

Аутентифікація через форму входу
-------------------------------------------------------------

Якщо ви спробуєте отримати доступ до адміністративного розділу, то будете перенаправлені на сторінку входу, де буде запропоновано ввести логін і пароль:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Увійдіть в систему, використовуючи логін ``admin`` і будь-який звичайний текстовий пароль, який ви закодували раніше. Якщо ви повністю скопіювали мою SQL-команду, пароль буде ``admin``.

Зверніть увагу, що EasyAdmin автоматично розпізнає систему аутентифікації Symfony:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Спробуйте натиснути на посилання "Sign out". Готово! Тепер у вас є повністю захищена панель керування.

.. index::
    single: Command;make:registration-form

.. note::

    Якщо ви хочете створити повнофункціональну систему аутентифікації з використанням форми, перегляньте команду ``make:registration-form``

.. sidebar:: Йдемо далі

    * `Документація по Symfony Security <https://symfony.com/doc/current/security.html>`_;

    * `Навчальний посібник SymfonyCasts: безпека <https://symfonycasts.com/screencast/symfony-security>`_;

    * `Як створити форму входу <https://symfony.com/doc/current/security/form_login_setup.html>`_ в застосунках Symfony;

    * `Шпаргалка з безпеки в Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
