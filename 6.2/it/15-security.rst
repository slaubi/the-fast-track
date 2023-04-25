Mettere in sicurezza il pannello amministrativo
===============================================

.. index::
    single: Components;Security
    single: Security

L'interfaccia del pannello amministrativo dovrebbe essere accessibile solo da persone autorizzate. La sicurezza di quest'area del sito può essere garantita usando il componente Security di Symfony.

Definire un'entity "User"
-------------------------

Anche se i partecipanti non saranno in grado di creare i propri account sul sito, creeremo un sistema di autenticazione completamente funzionante per l'amministratore del sito e avremo, quindi, un solo utente.

Il primo passo è quello di definire un'entity ``User``. Per evitare confusione, chiamiamola ``Admin``.

Per integrarsi con il sistema di autenticazione,  l'entity ``Admin`` deve seguire alcuni requisiti specifici. Per esempio, ha bisogno di una proprietà ``password``.

.. index::
    single: Command;make:user

Utilizzare il comando dedicato ``make:user`` per creare l'entity ``Admin`` al posto del tradizionale ``make:entity``:

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Rispondere alle domande interattive: "we want to use Doctrine to store the admins" (``yes``), "use ``username`` for the unique display name of admins", e "each user will have a password" (``yes``).

La classe generata contiene metodi come ``getRoles()``, ``eraseCredentials()`` e alcuni altri metodi che sono necessari al sistema di autenticazione di Symfony.

Se si desidera aggiungere altre proprietà all'utente ``Admin``, utilizzare il comando ``make:entity``.

Oltre a generare l'entity ``Admin``, il comando ha anche aggiornato la configurazione di sicurezza per collegare l'entity con il sistema di autenticazione:

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

Lasciamo a Symfony la scelta del miglior algoritmo possibile per l'hash delle password (che si evolverà nel tempo).

È giunto il momento di generare gli script di migrazione e migrare il database:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Generare una password per l'utente amministratore
-------------------------------------------------

.. index::
    single: Security;Password Hashes

Non svilupperemo un sistema dedicato per la creazione di account amministrativi. Come detto precedentemente, avremo sempre un solo amministratore. Il nome utente sarà ``admin`` e dobbiamo generare l'hash della password.

.. index::
    single: Command;security:hash-password

Selezionare ``App\Entity\Admin`` e scegliere la password desiderata, poi lanciare il seguente comando per generare l'hash della password:

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

Creare un amministratore
------------------------

.. index::
    single: Symfony CLI;run psql

Inserire l'utente amministratore tramite un'istruzione SQL:

.. code-block:: terminal

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Si noti l'escape del simbolo ``$`` nel valore della colonna della password;

Configurare l'autenticazione
----------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Ora che abbiamo un utente amministratore, possiamo proteggere il pannello amministrativo. Symfony supporta diverse strategie di autenticazione. Usiamo il classico e popolare *sistema di autenticazione con form*.

Eseguire il comando ``make:auth`` per aggiornare la configurazione, generare un template per il login e creare un *authenticator*:

.. code-block:: terminal
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Selezionare ``1`` per generare un'autenticazione con form, chiamare la classe authenticator ``AppAuthenticator``, il controller ``SecurityController``, e generare un URL ``/logout`` (``yes``).

Il comando ha aggiornato la configurazione di sicurezza per collegare le classi generate:

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

Come suggerito dall'output del comando, abbiamo bisogno di personalizzare la rotta nel metodo ``onAuthenticationSuccess()`` per reindirizzare l'utente quando accede:

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

    Come faccio a ricordare che il percorso per EasyAdmin è ``admin`` (in base alla configurazione in ``App\Controller\Admin\DashboardController``)? Non occorre ricordarselo. Possiamo dare un'occhiata al file, ma possiamo anche eseguire il seguente comando,  che mostra l'associazione tra i nomi delle rotte e i percorsi:

    .. code-block:: terminal

        $ symfony console debug:router

Aggiungere regole di accesso e autorizzazione
---------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Un sistema di sicurezza è composto da due parti: *autenticazione* e *autorizzazione*. Quando abbiamo creato l'utente amministratore, gli abbiamo assegnato il ruolo ``ROLE_ADMIN``. Limitiamo la sezione ``/admin`` solo agli utenti che hanno questo ruolo aggiungendo una regola a ``access_control``:

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

Le regole in ``access_control`` limitano l'accesso usando delle espressioni regolari. Quando si tenta di accedere a un URL che inizia con ``/admin``, il sistema di sicurezza verificherà se l'utente connesso abbia il ruolo ``ROLE_ADMIN``.

Autenticazione tramite form di login
------------------------------------

Se si tenta di accedere al pannello amministrativo, si dovrebbe ora essere reindirizzati alla pagina di login, dove sono richiesti nome utente e password:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Effettuare il login utilizzando ``admin`` e come password quella in chiaro che è stata precedentemente scelta. Se avete copiato esattamente il precedente comando SQL, la password è ``admin``.

Notare che EasyAdmin riconosce automaticamente il sistema di autenticazione di Symfony:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Provare a cliccare sul link "Sign out". Ecco fatto! Un pannello amministrativo completamente al sicuro.

.. index::
    single: Command;make:registration-form

.. note::

    Se si vuole creare un sistema completo di autenticazione con form, dare uno sguardo al comando ``make:registration-form``.

.. sidebar:: Andare oltre

    * `Documentazione di Symfony Security`_;

    * `Guida alla sicurezza su SymfonyCasts`_;

    * `Come costruire un form di login`_ nelle applicazioni Symfony;

    * `Cheat Sheet di Symfony Security`_.

.. _`Documentazione di Symfony Security`: https://symfony.com/doc/current/security.html
.. _`Guida alla sicurezza su SymfonyCasts`: https://symfonycasts.com/screencast/symfony-security
.. _`Come costruire un form di login`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`Cheat Sheet di Symfony Security`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
