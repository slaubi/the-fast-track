Захист панелі керування
============================================

.. index::
    single: Components;Security
    single: Security

Внутрішній інтерфейс панелі керування має бути доступний тільки довіреним особам. Захист цієї області веб-сайту може бути виконана за допомогою компонента Symfony Security.

Визначення сутності користувача
------------------------------------------------------------

Попри те що учасники не зможуть створити свої власні облікові записи на веб-сайті, ми створимо повнофункціональну систему аутентифікації для адміністратора. Тому у нас буде тільки один користувач — адміністратор веб-сайту.

Першим кроком є визначення сутності ``User``. Щоб уникнути можливої плутанини, назвімо її ``Admin``.

Щоб інтегрувати сутність ``Admin`` із системою аутентифікації Symfony Security, вона має відповідати певним вимогам. Наприклад, має бути властивість ``password``.

.. index::
    single: Command;make:user

Використовуйте спеціальну команду ``make:user``, щоб створити сутність ``Admin``, замість традиційної ``make:entity``:

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Дайте відповідь на інтерактивні запитання: ми хочемо використовувати Doctrine для зберігання адміністраторів (``yes``), використовувати властивість ``username`` для відображення унікального імені адміністратору, та чи кожен користувач матиме пароль (``yes``).

Згенерований клас містить такі методи, як ``getRoles()``, ``eraseCredentials()`` і деякі інші, які необхідні для системи аутентифікації Symfony.

Якщо ви хочете додати додаткові властивості до користувача ``Admin`` — використовуйте команду ``make:entity``.

На додаток до генерування сутності ``Admin``, команда також оновила конфігурацію безпеки, щоб зв'язати сутність із системою аутентифікації:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 11,12,20

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -5,14 +5,18 @@ security:
             Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
         # https://symfony.com/doc/current/security.html#loading-the-user-the-user-provider
         providers:
    -        users_in_memory: { memory: null }
    +        # used to reload user from session & other features (e.g. switch_user)
    +        app_user_provider:
    +            entity:
    +                class: App\Entity\Admin
    +                property: username
         firewalls:
             dev:
                 pattern: ^/(_(profiler|wdt)|css|images|js)/
                 security: false
             main:
                 lazy: true
    -            provider: users_in_memory
    +            provider: app_user_provider

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#the-firewall

Ми дозволяємо Symfony вибрати найкращий з можливих алгоритмів хешування паролів (який буде розвиватися з часом).

Настав час згенерувати міграцію та застосувати її до бази даних:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Генерування пароля для адміністратора
-----------------------------------------------------------------------

.. index::
    single: Security;Password Hashes

Ми не будемо розробляти окрему систему для створення облікових записів адміністраторів. Знову ж таки, у нас буде тільки один адміністратор. Логіном буде ``admin``, і нам потрібно згенерувати хеш пароля.

.. index::
    single: Command;security:hash-password

Виберіть ``App\Entity\Admin``, а потім виберіть потрібний пароль і виконайте наступну команду, щоб згенерувати хеш пароля:

.. code-block:: terminal
    :class: answers(0||admin)

    $ symfony console security:hash-password

.. code-block:: text
    :class: ignore
    :emphasize-lines: 11

    Symfony Password Hash Utility
    =============================

     Type in your password to be hashed:
     >

     ------------------ ---------------------------------------------------------------------------------------------------
      Key                Value
     ------------------ ---------------------------------------------------------------------------------------------------
      Hasher used        Symfony\Component\PasswordHasher\Hasher\MigratingPasswordHasher
      Password hash      $argon2id$v=19$m=65536,t=4,p=1$BQG+jovPcunctc30xG5PxQ$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s
     ------------------ ---------------------------------------------------------------------------------------------------

     ! [NOTE] Self-salting hasher used: the hasher generated its own built-in salt.


     [OK] Password hashing succeeded

Створення адміністратора
-----------------------------------------------

.. index::
    single: Symfony CLI;run psql

Додайте користувача-адміністратора за допомогою SQL-виразу:

.. code-block:: terminal

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

.. code-block:: terminal
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Виберіть ``1``, щоб згенерувати аутентифікатор форми входу, назвіть клас аутентифікатора — ``AppAuthenticator``, контролер — ``SecurityController`` і згенеруйте URL-адресу ``/logout`` (``yes``).

Команда оновила конфігурацію безпеки для налаштування згенерованих класів:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 9

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -17,6 +17,11 @@ security:
             main:
                 lazy: true
                 provider: app_user_provider
    +            custom_authenticator: App\Security\AppAuthenticator
    +            logout:
    +                path: app_logout
    +                # where to redirect after logout
    +                # target: app_any_route

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#the-firewall

Завдяки підказці, що з'явилася після виконання команди, можна зрозуміти, що нам потрібно налаштувати маршрут у методі ``onAuthenticationSuccess()``, щоб переспрямувати користувача після успішного входу в систему:

.. code-block:: diff

    --- a/src/Security/AppAuthenticator.php
    +++ b/src/Security/AppAuthenticator.php
    @@ -46,9 +46,7 @@ class AppAuthenticator extends AbstractLoginFormAuthenticator
                 return new RedirectResponse($targetPath);
             }

    -        // For example:
    -        // return new RedirectResponse($this->urlGenerator->generate('some_route'));
    -        throw new \Exception('TODO: provide a valid redirect inside '.__FILE__);
    +        return new RedirectResponse($this->urlGenerator->generate('admin'));
         }

         protected function getLoginUrl(Request $request): string

.. index::
    single: Command;debug:router
    single: Routing;Debug
    single: Debug;Routing

.. tip::

    Як я пам'ятаю, що маршрутом EasyAdmin є ``admin`` (як налаштовано в ``App\Controller\Admin\DashboardController``)? Я не пам'ятаю. Ви можете поглянути на файл, але також можна виконати наступну команду, яка показує зв'язок між іменами маршрутів і шляхами:

    .. code-block:: terminal

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
    @@ -31,7 +31,7 @@ security:
         # Easy way to control access for large sections of your site
         # Note: Only the *first* access control that matches will be used
         access_control:
    -        # - { path: ^/admin, roles: ROLE_ADMIN }
    +        - { path: ^/admin, roles: ROLE_ADMIN }
             # - { path: ^/profile, roles: ROLE_USER }

     when@test:

Правила ``access_control`` обмежують доступ за допомогою регулярних виразів. При спробі переходу за URL-адресою, що починається з ``/admin``, система безпеки перевірить наявність ролі ``ROLE_ADMIN`` у користувача, що здійснив вхід.

Аутентифікація через форму входу
-------------------------------------------------------------

Якщо ви спробуєте отримати доступ до адміністративного розділу, то будете перенаправлені на сторінку входу, де буде запропоновано ввести логін і пароль:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Увійдіть у систему, використовуючи логін ``admin`` і будь-який текстовий пароль, який ви вибрали раніше. Якщо ви повністю скопіювали мою SQL-команду, паролем буде ``admin``.

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

    * `Документація по Symfony Security`_;

    * `Навчальний посібник SymfonyCasts: безпека`_;

    * `Як створити форму входу`_ в застосунках Symfony;

    * `Шпаргалка з безпеки в Symfony`_.

.. _`Документація по Symfony Security`: https://symfony.com/doc/current/security.html
.. _`Навчальний посібник SymfonyCasts: безпека`: https://symfonycasts.com/screencast/symfony-security
.. _`Як створити форму входу`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`Шпаргалка з безпеки в Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
