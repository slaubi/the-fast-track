Securizarea Backend-ului Admin
==============================

Interfața admin backend trebuie să fie accesibilă numai de către persoane de încredere. Securizarea acestei zone a site-ului poate fi făcută folosind componenta Symfony Security.

Ca și pentru Twig, componenta de securitate este deja instalată prin intermediul dependențelor tranzitive. Să o adăugăm explicit în fișierul ``composer.json`` al proiectului:

.. index::
    single: Components;Security
    single: Security

.. code-block:: bash

    $ symfony composer req security

Definirea unei entități de utilizator
---------------------------------------

Chiar dacă participanții nu își vor putea crea propriile conturi pe site-ul web, vom crea un sistem de autentificare complet funcțional pentru administrator. Prin urmare, vom avea un singur utilizator, administratorul site-ului.

Primul pas este definirea unei entități ``User``. Pentru a evita confuziile, îl numim ``Admin``.

Pentru a integra entitatea ``Admin`` cu sistemul de autentificare Symfony Security, trebuie să respecte anumite cerințe specifice. De exemplu, are nevoie de o proprietate ``password``.

.. index::
    single: Command;make:user

Folosește comanda dedicată ``make:user`` pentru a crea entitatea ``Admin`` în loc de tradiționala ``make:entity``:

.. code-block:: bash
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Răspunde la întrebările interactive: dorim să folosim Doctrine pentru a stoca administratorii (``yes``), să utilizăm un ``username`` pentru afișarea numelui unic al administratorilor și fiecare utilizator va avea o parolă (``yes``) .

Clasa generată conține metode precum ``getRoles ()``, ``eraseCredentials()`` și alte câteva care sunt necesare pentru sistemul de autentificare Symfony.

Dacă dorești să adaugi mai multe proprietăți utilizatorului ``Admin``, folosește ``make:entity``.

Să adăugăm o metodă ``__toString()``, așa cum dorește EasyAdmin:

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

Pe lângă generarea entității ``Admin``, comanda a actualizat și configurația de securitate pentru a conecta entitatea cu sistemul de autentificare:

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

Lăsăm Symfony să selecteze cel mai bun algoritm posibil pentru codificarea parolelor (care va evolua în timp).

A sosit momentul generării unei migrări și migrarea bazei de date:

.. code-block:: bash

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Generarea unei parole pentru utilizatorul cu rol de administrator
-----------------------------------------------------------------

.. index::
    single: Security;Encoding Passwords

Nu vom dezvolta un sistem dedicat pentru a crea conturi de admin. Din nou, vom avea un singur administrator. Login-ul va fi ``admin`` și trebuie să codificăm parola.

.. index::
    single: Command;security:encode-password

Alege orice dorești ca parolă și execută următoarea comandă pentru a genera parola codificată:

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

Crearea unui administrator
--------------------------

.. index::
    single: Symfony CLI;run psql

Introdu utilizatorul admin printr-o instrucțiune SQL:

.. code-block:: bash

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Observă ignorarea simbolului ``$`` în valoarea coloanei de parole; ignoră-le pe toate!

Configurarea autentificării de securitate
------------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Acum că avem un utilizator admin, putem securiza backend-ul admin. Symfony acceptă mai multe strategii de autentificare. Să folosim un sistem clasic și popular *de formular de autentificare*.

Execută ``make:auth`` pentru actualizarea configurației de securitate, generarea unui șablon de autentificare și crearea unui * autentificator*:

.. code-block:: bash
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Selectează ``1`` pentru a genera un formular de autentificare, denumește clasa de autentificator ``AppAuthenticator``, controlerul ``SecurityController`` și generează un URL ``/logout`` (``yes``).

Comanda a actualizat configurația de securitate pentru a conecta clasele generate:

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

După cum sugerează ieșirea comenzii, trebuie să personalizăm ruta în metoda ``onAuthenticationSuccess()`` pentru a redirecționa utilizatorul atunci când se conectează cu succes:

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

    De unde știu că ruta EasyAdmin este ``easyadmin``? Nu știu. Dar am executat următoarea comandă care arată asocierea între numele traseelor și cărări:

    .. code-block:: bash

        $ symfony console debug:router

Adăugarea regulilor de control de acces pentru autorizare
----------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Un sistem de securitate este format din două părți: *autentificare* și *autorizare*. La crearea utilizatorului de admin, le-am dat rolul ``ROLE_ADMIN``. Să restricționăm secțiunea ``/admin`` la utilizatorii care au acest rol adăugând o regulă la ``access_control``:

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

Regulile ``access_control`` restricționează accesul prin expresii regulate. Când încerci să accesezi o adresă URL care începe cu ``/admin``, sistemul de securitate va verifica dacă utilizatorul conectat dispune de rolul ``ROLE_ADMIN``.

Autentificare prin formularul de logare
---------------------------------------

Dacă încerci să accesezi backend-ul admin, trebuie să fii redirecționat la pagina de logare în care să introduci un login și o parolă:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Conectează-te folosind ``admin`` și orice parolă cu text simplu pe care ai codificat-o anterior. Dacă ai copiat exact comanda SQL, parola este ``admin``.

Reține că EasyAdmin recunoaște automat sistemul de autentificare Symfony:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Încearcă să faci clic pe linkul „Sign out”. Îl ai! Un backend de securizare complet.

.. index::
    single: Command;make:registration-form

.. note::

    Dacă dorești să creezi un sistem complet de formulare de autentificare, aruncă o privire la comanda 'make:registration-form'.

.. sidebar:: Mergând mai departe

    * Documentația `Symfony Security <https://symfony.com/doc/current/security.html>`_;

    * `Tutorial SymfonyCasts Security <https://symfonycasts.com/screencast/symfony-security>`_;

    * `Cum se creează un formular de autentificare <https://symfony.com/doc/current/security/form_login_setup.html>`_ în aplicațiile Symfony;

    * `Notițe Symfony Security <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
