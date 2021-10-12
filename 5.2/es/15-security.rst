Asegurando el panel de administración
=====================================

La interfaz del panel de administración sólo debe ser accesible para personas de confianza. El control de acceso a este área del sitio web se puede realizar utilizando el componente Symfony Security.

Al igual que para Twig, el componente de seguridad ya está instalado a través de dependencias transitivas. Añadámoslo explícitamente al archivo ``composer.json`` del proyecto:

.. index::
    single: Components;Security
    single: Security

.. code-block:: bash

    $ symfony composer req security

Definiendo una entidad User
---------------------------

Aunque los asistentes no puedan crear sus propias cuentas en el sitio web, vamos a crear un sistema de autenticación completamente funcional para el administrador. Por lo tanto, sólo tendremos un usuario, el administrador del sitio web.

El primer paso es definir una entidad ``User``. Para evitar confusiones, vamos a llamarla ``Admin``.

Para que la entidad ``Admin`` pueda integrarse con el sistema de autenticación Symfony Security, necesita cumplir con algunos requisitos específicos. Por ejemplo, necesita contar con una propiedad ``password``.

.. index::
    single: Command;make:user

Utiliza el comando dedicado ``make:user`` en lugar del tradicional ``make:entity`` para crear la entidad ``Admin``:

.. code-block:: bash
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Responde a las preguntas interactivas: queremos usar Doctrine para almacenar los admins (``yes``), ``username`` para el nombre de pantalla único de los admins, y cada usuario tendrá una contraseña (``yes``).

La clase generada contiene métodos como ``getRoles()``, ``eraseCredentials()``, y otros pocos que son necesarios para el sistema de autenticación de Symfony.

Si deseas añadir más propiedades al usuario ``Admin``, utiliza ``make:entity`` .

Añadamos un método ``__toString()`` como le gusta a EasyAdmin:

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

Además de generar la entidad ``Admin``, el comando también actualizó la configuración de seguridad para conectar la entidad con el sistema de autenticación:

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

Dejamos que Symfony seleccione el mejor algoritmo posible para la codificación de contraseñas (el cuál evolucionará con el tiempo).

Hora de generar una migración y migrar la base de datos:

.. code-block:: bash

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Generando una contraseña para el usuario administrador
------------------------------------------------------

.. index::
    single: Security;Encoding Passwords

No desarrollaremos un sistema dedicado para crear cuentas de administración. Una vez más, sólo tendremos un administrador. El login será ``admin`` y necesitamos codificar la contraseña.

.. index::
    single: Command;security:encode-password

Elige lo que quieras como contraseña y ejecuta el siguiente comando para generar la contraseña codificada:

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

Creando un administrador
------------------------

.. index::
    single: Symfony CLI;run psql

Inserta el usuario administrador a través de una sentencia SQL:

.. code-block:: bash

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Observa el carácter de escape en el signo ``$`` que hay en la columna de contraseña; ¡usa secuencias de escape en todos!

Configurando la autenticación de seguridad
------------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Ahora que tenemos un usuario administrador, podemos asegurar el panel de administración. Symfony permite varias estrategias de autenticación. Usemos un *sistema de autenticación de formularios* clásico y popular.

Ejecuta el comando ``make:auth`` para actualizar la configuración de seguridad, generar una plantilla de inicio de sesión y crear un *autenticador* :

.. code-block:: bash
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Selecciona ``1`` para generar un autenticador de formulario de inicio de sesión, nombra la clase autenticador ``AppAuthenticator``, el controlador ``SecurityController``, y generar una URL ``/logout`` (``yes``).

El comando actualizó la configuración de seguridad para conectar las clases generadas:

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

Como indica la salida del comando, necesitamos personalizar la ruta en el método ``onAuthenticationSuccess()`` para redirigir al usuario cuando inicie sesión con éxito:

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

    ¿Cómo recuerdo que la ruta de EasyAdmin es ``admin`` (como configuramos en ``App\Controller\Admin\DashboardController``)? No lo sé. Puedes echar un vistazo al archivo, pero también puedes ejecutar el siguiente comando que muestra la asociación entre los nombres de ruta y las rutas:

    .. code-block:: bash

        $ symfony console debug:router

Añadiendo reglas de autorización de control de acceso
-----------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Un sistema de seguridad consta de dos partes: *autenticación* y *autorización*. Cuando creamos el usuario admin, le dimos el rol ``ROLE_ADMIN``. Vamos a restringir la sección ``/admin`` a los usuarios que tengan esta función añadiendo una regla a ``access_control``:

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

Las reglas ``access_control`` restringen el acceso mediante expresiones regulares. Cuando se intenta acceder a una URL que comienza con ``/admin``, el sistema de seguridad comprobará que el usuario que ha iniciado sesión tenga asignado el rol ``ROLE_ADMIN``.

Autenticación a través del formulario de inicio de sesión
---------------------------------------------------------

Si intentas acceder al módulo de administración, serás redirigido a la página de inicio de sesión y se te pedirá que introduzcas un nombre de usuario y una contraseña:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Inicia sesión usando ``admin`` y cualquier contraseña de texto plano que hayas codificado anteriormente. Si copiaste mi comando SQL exactamente, la contraseña es ``admin`` .

Ten en cuenta que EasyAdmin reconoce automáticamente el sistema de autenticación de Symfony:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Intenta hacer clic en el enlace "Cerrar sesión". ¡Lo tienes! Un panel de administración totalmente seguro.

.. index::
    single: Command;make:registration-form

.. note::

    Si deseas crear un sistema de autenticación de formularios completo, echa un vistazo al comando ``make:registration-form``.

.. sidebar:: Yendo más allá

    * La `documentación de Symfony Security <https://symfony.com/doc/current/security.html>`_;

    * `Tutorial de SymfonyCasts Security <https://symfonycasts.com/screencast/symfony-security>`_ ;

    * `Cómo crear un formulario de inicio de sesión <https://symfony.com/doc/current/security/form_login_setup.html>`_ en aplicaciones Symfony;

    * La `Chuleta de Symfony Security <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
