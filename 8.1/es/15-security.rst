Asegurando el panel de administración
======================================

La interfaz del panel de administración sólo debe ser accesible para personas de confianza. El control de acceso a este área del sitio web se puede realizar utilizando el componente Symfony Security.

.. index::
    single: Components;Security
    single: Security

Definiendo una entidad User
---------------------------

Aunque los asistentes no puedan crear sus propias cuentas en el sitio web, vamos a crear un sistema de autenticación completamente funcional para el administrador. Por lo tanto, sólo tendremos un usuario, el administrador del sitio web.

El primer paso es definir una entidad ``User``. Para evitar confusiones, vamos a llamarla ``Admin``.

Para que la entidad ``Admin`` pueda integrarse con el sistema de autenticación Symfony Security, necesita cumplir con algunos requisitos específicos. Por ejemplo, necesita contar con una propiedad ``password``.

.. index::
    single: Command;make:user

Utiliza el comando dedicado ``make:user`` en lugar del tradicional ``make:entity`` para crear la entidad ``Admin``:

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Responde a las preguntas interactivas: queremos usar Doctrine para almacenar los admins (``yes``), ``username`` para el nombre de pantalla único de los admins, y cada usuario tendrá una contraseña (``yes``).

La clase generada contiene métodos como ``getRoles()``, ``eraseCredentials()``, y otros pocos que son necesarios para el sistema de autenticación de Symfony.

Si deseas añadir más propiedades al usuario ``Admin``, utiliza ``make:entity`` .

Además de generar la entidad ``Admin``, el comando también actualizó la configuración de seguridad para conectar la entidad con el sistema de autenticación:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 11,12,20

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
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

Dejamos que Symfony seleccione el mejor algoritmo posible para generar el hash de las contraseñas (el cuál evolucionará con el tiempo).

Hora de generar una migración y migrar la base de datos:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Generando una contraseña para el usuario administrador
-------------------------------------------------------

.. index::
    single: Security;Password Hashes

No desarrollaremos un sistema dedicado para crear cuentas de administración. Una vez más, sólo tendremos un administrador. El login será ``admin`` y necesitamos generar el hash de la contraseña.

.. index::
    single: Command;security:hash-password

Elige lo que quieras como contraseña y ejecuta el siguiente comando para generar el hash de la contraseña:

.. code-block:: terminal
    :class: answers(admin)

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

Creando un administrador
------------------------

.. index::
    single: Symfony CLI;run psql

Inserta el usuario administrador a través de una sentencia SQL:

.. code-block:: terminal

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Observa el carácter de escape en el signo ``$`` que hay en la columna de contraseña; ¡usa secuencias de escape en todos!

Configurando la autenticación de seguridad
-------------------------------------------

.. index::
    single: Command;make:security:form-login
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Ahora que tenemos un usuario administrador, podemos asegurar el panel de administración. Symfony permite varias estrategias de autenticación. Usemos un *sistema de autenticación de formularios* clásico y popular.

Ejecuta el comando ``make:security:form-login`` para actualizar la configuración de seguridad, generar una plantilla de inicio de sesión y crear un *autenticador* :

.. code-block:: terminal
    :class: answers(SecurityController||yes)

    $ symfony console make:security:form-login

Nombra el controlador ``SecurityController`` y genera una URL ``/logout`` (``yes``).

El comando actualizó la configuración de seguridad para conectar las clases generadas:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 9

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -15,7 +15,15 @@ security:
                 security: false
             main:
                 lazy: true
    -            provider: users_in_memory
    +            provider: app_user_provider
    +            form_login:
    +                login_path: app_login
    +                check_path: app_login
    +                enable_csrf: true
    +            logout:
    +                path: app_logout
    +                # where to redirect after logout
    +                # target: app_any_route

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#the-firewall

.. index::
    single: Command;debug:router
    single: Routing;Debug
    single: Debug;Routing

.. tip::

    ¿Cómo recuerdo que la ruta de EasyAdmin es ``admin`` (como configuramos en ``App\Controller\Admin\DashboardController``)? No lo sé. Puedes echar un vistazo al archivo, pero también puedes ejecutar el siguiente comando que muestra la asociación entre los nombres de ruta y las rutas:

    .. code-block:: terminal

        $ symfony console debug:router

Añadiendo reglas de autorización de control de acceso
-------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Un sistema de seguridad consta de dos partes: *autenticación* y *autorización*. Cuando creamos el usuario admin, le dimos el rol ``ROLE_ADMIN``. Vamos a restringir la sección ``/admin`` a los usuarios que tengan esta función añadiendo una regla a ``access_control``:

.. code-block:: diff
    :emphasize-lines: 8

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -34,7 +34,7 @@ security:
         # Easy way to control access for large sections of your site
         # Note: Only the *first* access control that matches will be used
         access_control:
    -        # - { path: ^/admin, roles: ROLE_ADMIN }
    +        - { path: ^/admin, roles: ROLE_ADMIN }
             # - { path: ^/profile, roles: ROLE_USER }

     when@test:

Las reglas ``access_control`` restringen el acceso mediante expresiones regulares. Cuando se intenta acceder a una URL que comienza con ``/admin``, el sistema de seguridad comprobará que el usuario que ha iniciado sesión tenga asignado el rol ``ROLE_ADMIN``.

Autenticación a través del formulario de inicio de sesión
------------------------------------------------------------

Si intentas acceder al módulo de administración, serás redirigido a la página de inicio de sesión y se te pedirá que introduzcas un nombre de usuario y una contraseña:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Inicia sesión usando ``admin`` y cualquier contraseña de texto plano que hayas elegido anteriormente. Si copiaste mi comando SQL exactamente, la contraseña es ``admin`` .

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

    * La `documentación de Symfony Security`_ ;

    * `Tutorial de SymfonyCasts Security`_ ;

    * `Cómo crear un formulario de inicio de sesión`_ en aplicaciones Symfony;

    * La `Chuleta de Symfony Security`_ .

.. _`documentación de Symfony Security`: https://symfony.com/doc/current/security.html
.. _`Tutorial de SymfonyCasts Security`: https://symfonycasts.com/screencast/symfony-security
.. _`Cómo crear un formulario de inicio de sesión`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`Chuleta de Symfony Security`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
