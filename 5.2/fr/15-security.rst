Sécuriser l'interface d'administration
=======================================

L'interface d'administration ne doit être accessible que par des personnes autorisées. La sécurisation de cette zone du site peut se faire à l'aide du composant Symfony Security.

Comme pour Twig, le composant de sécurité est déjà installé par des dépendances transitives. Ajoutons-le explicitement au fichier ``composer.json`` du projet :

.. index::
    single: Components;Security
    single: Security

.. code-block:: bash

    $ symfony composer req security

Définir une entité User
-------------------------

Même si les internautes ne pourront pas créer leur propre compte sur le site, nous allons créer un système d'authentification entièrement fonctionnel pour l'admin. Nous n'aurons donc qu'un seul User, l'admin du site.

La première étape consiste à définir une entité ``User``. Pour éviter toute confusion, nommons-la plutôt ``Admin``.

Pour utiliser l'entité ``Admin`` dans le système d'authentification de Symfony, celle-ci doit respecter certaines exigences spécifiques. Par exemple, elle a besoin d'une propriété ``password``.

.. index::
    single: Command;make:user

Utilisez la commande dédiée ``make:user`` pour créer l'entité ``Admin``  au lieu de la commande traditionnelle ``make:entity`` :

.. code-block:: bash
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

Répondez aux questions qui vous sont posées : nous voulons utiliser Doctrine pour stocker nos users (``yes``), utiliser ``username`` pour le nom d'affichage unique des admins et chaque admin aura un mot de passe (``yes``).

La classe générée contient des méthodes comme ``getRoles()``, ``eraseCredentials()`` et d'autres qui sont nécessaires au système d'authentification de Symfony.

Si vous voulez ajouter d'autres propriétés à l'entité ``Admin``, exécutez ``make:entity``.

Ajoutons une méthode ``__toString()`` comme EasyAdmin les aime :

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

En plus de générer l'entité ``Admin``, la commande a également mis à jour la configuration de sécurité pour connecter l'entité au système d'authentification :

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

Nous laissons Symfony choisir le meilleur algorithme possible pour encoder les mots de passe (il évoluera avec le temps).

Il est temps de générer une migration et de migrer la base de données :

.. code-block:: bash

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

Générer un mot de passe pour l'admin
--------------------------------------

.. index::
    single: Security;Encoding Passwords

Nous ne développerons pas de système dédié pour créer des comptes d'administration. Encore une fois, nous n'aurons qu'un seul admin. Le login sera ``admin`` et nous devons encoder le mot de passe.

.. index::
    single: Command;security:encode-password

Choisissez ce que vous voulez comme mot de passe et exécutez la commande suivante pour générer le mot de passe encodé :

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

Créer un admininistrateur
--------------------------

.. index::
    single: Symfony CLI;run psql

Insérez l'admin grâce à une requête SQL :

.. code-block:: bash

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

Notez l'échappement du caractère ``$`` dans le mot de passe ; échappez tous les caractères qui en ont besoin !

Configurer le système d'authentification
-----------------------------------------

.. index::
    single: Command;make:auth
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

Maintenant que nous avons un admin, nous pouvons sécuriser l'interface d'administration. Symfony accepte plusieurs stratégies d'authentification. Utilisons un classique *système d'authentification par formulaire*.

Exécutez la commande ``make:auth`` pour mettre à jour la configuration de sécurité, générer un template pour la connexion et créer une classe d'authentification (*authenticator*) :

.. code-block:: bash
    :class: answers(1||AppAuthenticator||SecurityController||yes)

    $ symfony console make:auth

Sélectionnez ``1`` pour générer une classe d'authentification pour le  formulaire de connexion, nommez la classe d'authentification ``AppAuthenticator``, le contrôleur ``SecurityController`` et créez une URL ``/logout`` (``yes``).

La commande a mis à jour la configuration de sécurité pour lier les classes générées :

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

Comme l'indique la sortie de la commande, nous devons personnaliser la route dans la méthode ``onAuthenticationSuccess()`` pour rediriger l'admin lorsqu'il a réussi à se connecter :

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

    Comment puis-je savoir que la route d'EasyAdmin est ``admin`` (comme spécifié dans ``App\Controller\Admin\DashboardController``) ? Je ne peux pas. Mais j'ai lancé la commande suivante qui montre l'association entre les noms de route et les chemins :

    .. code-block:: bash

        $ symfony console debug:router

Ajouter les règles de contrôle d'accès
-----------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

Un système de sécurité se compose de deux parties : l'*authentification* et l'*autorisation*. Lors de la création de l'admin, nous lui avons donné le rôle ``ROLE_ADMIN``. Limitons la section ``/admin`` aux seules personnes ayant ce rôle en ajoutant une règle à ``access_control`` :

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

Les règles ``access_control`` limitent l'accès par des expressions régulières. Lorsqu'une personne connectée tente d'accéder à une URL qui commence par ``/admin``, le système de sécurité vérifie qu'elle a bien le rôle ``ROLE_ADMIN``.

S'authentifier avec le formulaire de connexion
----------------------------------------------

Si vous essayez d'accéder à l'interface d'administration, vous devriez maintenant être redirigé vers la page de connexion et être invité à entrer un identifiant et un mot de passe :

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

Connectez-vous en utilisant ``admin`` et le mot de passe que vous avez encodé précédemment. Si vous avez copié exactement ma requête SQL, le mot de passe est ``admin``.

Notez qu'EasyAdmin s'intègre automatiquement au système d'authentification de Symfony :

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

Essayez de cliquer sur le lien "Sign out". Et voilà ! Nous avons une interface d'administration entièrement sécurisée.

.. index::
    single: Command;make:registration-form

.. note::

    Si vous voulez créer un système complet d'authentification par formulaire, jetez un coup d’œil à la commande ``make:registration-form``.

.. sidebar:: Aller plus loin

    * La `documentation de la sécurité de Symfony <https://symfony.com/doc/current/security.html>`_ ;

    * `Tutoriel SymfonyCasts sur la sécurité <https://symfonycasts.com/screencast/symfony-security>`_ ;

    * `Comment créer un formulaire de connexion <https://symfony.com/doc/current/security/form_login_setup.html>`_ dans les applications Symfony ;

    * La `cheat sheet de la sécurité dans Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf>`_.
