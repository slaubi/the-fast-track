Securing the Admin Backend
==========================

.. index::
    single: Components;Security
    single: Security

The admin backend interface should only be accessible by trusted people. Securing this area of the website can be done using the Symfony Security component.

Defining a User Entity
----------------------

Even if attendees won't be able to create their own accounts on the website, we are going to create a fully functional authentication system for the admin. We will therefore only have one user, the website admin.

The first step is to define a ``User`` entity. To avoid any confusions, let's name it ``Admin`` instead.

To integrate the ``Admin`` entity with the Symfony Security authentication system, it needs to follow some specific requirements. For instance, it needs a ``password`` property.

.. index::
    single: Command;make:user

Use the dedicated ``make:user`` command to create the ``Admin`` entity instead of the traditional ``make:entity`` one:

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Answer the interactive questions: we want to use Doctrine to store the admins (``yes``), use ``username`` for the unique display name of admins, and each user will have a password (``yes``).

The generated class contains methods like ``getRoles()``, ``eraseCredentials()``, and a few others that are needed by the Symfony authentication system.

If you want to add more properties to the ``Admin`` user, use ``make:entity``.

In addition to generating the ``Admin`` entity, the command also updated the security configuration to wire the entity with the authentication system:

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

We let Symfony select the best possible algorithm for hashing passwords (which will evolve over time).

Time to generate a migration and migrate the database:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Generating a Password for the Admin User
----------------------------------------

.. index::
    single: Security;Password Hashes

We won't develop a dedicated system to create admin accounts. Again, we will only ever have one admin. The login will be ``admin`` and we need to generate the password hash.

.. index::
    single: Command;security:hash-password

Select ``App\Entity\Admin`` and then choose whatever you like as a password and run the following command to generate the password hash:

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

Creating an Admin
-----------------

.. index::
    single: Symfony CLI;run psql

Insert the admin user via an SQL statement:

.. code-block:: terminal

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Note the escaping of the ``$`` sign in the password column value; escape them all!

Configuring the Security Authentication
---------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Now that we have an admin user, we can secure the admin backend. Symfony supports several authentication strategies. Let's use a classic and popular *form authentication system*.

Run the ``make:auth`` command to update the security configuration, generate a login template, and create an *authenticator*:

.. code-block:: terminal
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Select ``1`` to generate a login form authenticator, name the authenticator class ``AppAuthenticator``, the controller ``SecurityController``, and generate a ``/logout`` URL (``yes``).

The command updated the security configuration to wire the generated classes:

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

As hinted by the command output, we need to customize the route in the ``onAuthenticationSuccess()`` method to redirect the user when they successfully sign in:

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

    How do I remember that the EasyAdmin route is ``admin`` (as configured in ``App\Controller\Admin\DashboardController``)? I don't. You can have a look at the file, but you can also run the following command that shows the association between route names and paths:

    .. code-block:: terminal

        $ symfony console debug:router

Adding Authorization Access Control Rules
-----------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

A security system is made of two parts: *authentication* and *authorization*. When creating the admin user, we gave them the ``ROLE_ADMIN`` role. Let's restrict the ``/admin`` section to users having this role by adding a rule to ``access_control``:

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

The ``access_control`` rules restrict access by regular expressions. When trying to access a URL that starts with ``/admin``, the security system will check for the ``ROLE_ADMIN`` role on the logged-in user.

Authenticating via the Login Form
---------------------------------

If you try to access the admin backend, you should now be redirected to the login page and prompted to enter a login and a password:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Log in using ``admin`` and whatever plain-text password you chose earlier. If you copied my SQL command exactly, the password is ``admin``.

Note that EasyAdmin automatically recognizes the Symfony authentication system:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Try to click on the "Sign out" link. You have it! A fully-secured backend admin.

.. index::
    single: Command;make:registration-form

.. note::

    If you want to create a fully-featured form authentication system, have a look at the ``make:registration-form`` command.

.. sidebar:: Going Further

    * The `Symfony Security docs`_;

    * `SymfonyCasts Security tutorial`_;

    * `How to Build a Login Form`_ in Symfony applications;

    * The `Symfony Security Cheat Sheet`_.

.. _`Symfony Security docs`: https://symfony.com/doc/current/security.html
.. _`SymfonyCasts Security tutorial`: https://symfonycasts.com/screencast/symfony-security
.. _`How to Build a Login Form`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`Symfony Security Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
