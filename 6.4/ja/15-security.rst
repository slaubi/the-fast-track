管理者のバックエンドをセキュアにする
======================================================

.. index::
    single: Components;Security
    single: Security

管理者のバックエンドのインターフェースは、信頼された人からのみアクセス可能であるべきです。Symfony のセキュリティコンポーネントを使用して、Web サイトをセキュアにします。

User エンティティを定義する
--------------------------------------

参加者が Web サイトに自分のアカウントを作成することはできないですが、ここでは管理者のために正しく機能する認証システムを作成しましょう。そのために、Webサイトの管理者のユーザーを一つだけ用意します。

最初のステップは、 ``User`` エンティティを定義することです。混乱を避けるためにここでは ``Admin`` を使います。

Symfony のセキュリティ認証システムで ``Admin`` エンティティを使用するためには、``password`` プロパティなどの要件が必要になります。

.. index::
    single: Command;make:user

``make:entity`` ではなく、専用の ``make:user`` コマンドを使用して ``Admin`` エンティティを作成してください:

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

インタラクティブな質問に次のように答えてください: 管理者を Doctrine に格納したいので (``yes``)、 管理者のユニークな表示名を ``username`` 、そして各ユーザーがパスワードを1つ持つことに(``yes``)と。

生成されたクラスには、``getRole()``, ``eraseCredentials()`` メソッドの他にも Symfony の認証システムで必要なものが入っています。

``Admin`` ユーザーにさらにプロパティを追加したければ、 ``make:entity`` を使用してください。

``Admin`` エンティティを生成するだけでなく、このコマンドは、認証システムとエンティティのワイヤリングのためのセキュリティ設定を更新します:

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

Symfony にパスワードをハッシュ化するのに一番有効なアルゴリズムを選択させましょう（これは時が経つと変更されていくものです）。

マイグレーションを生成して、データベースをmigrateします:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

管理者ユーザーのパスワードを生成する
------------------------------------------------------

.. index::
    single: Security;Password Hashes

管理者のアカウントを作成するのに専用のシステムを開発することはないです。ここでは1つの管理者しか用意しませんから。ログインするには ``admin`` とハッシュ化されたパスワードが必要になります。

.. index::
    single: Command;security:hash-password

``App\Entity\Admin`` を選び、好きなパスワードを選択して、次のコマンドを実行しパスワードをハッシュ化してください:

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

管理者を作成する
------------------------

.. index::
    single: Symfony CLI;run psql

次の SQL で管理者ユーザーを追加してください:

.. code-block:: terminal

    $ symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

パスワードの値の ``$`` 符号は全てエスケープしましょう。

セキュリティ認証を設定する
---------------------------------------

.. index::
    single: Command;make:security:form-login
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

管理者ユーザーができましたので、管理者のバックエンドをセキュアにすることができます。 Symfony は複数の認証の方法をサポートしていますが、ここでは、昔から人気のある *フォーム認証システム* を使いましょう。

``make:auth`` コマンドを実行しセキュリティ設定を更新し、ログインテンプレートを作成し、 *認証システム* を作成しましょう:

.. code-block:: terminal
    :class: answers(SecurityController||yes)

    $ symfony console make:security:form-login

``1`` を選択し、ログインフォーム認証システムを生成し、 ``AppAuthenticator`` とし、コントローラーを ``SecurityController``  と命名し、 ``logout`` URLを生成しましょう(``yes``)。

このコマンドはセキュリティ設定を更新し生成されるクラスとワイヤリングします:

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

    EasyAdminのルートが ``admin`` であることをどうやって覚えていられるでしょうか？( ``App\Controller\Admin\DashboardController`` で設定しました) 覚えていられませんよね。そんなときはコントローラファイルを見ることもできますが、次のコマンドでルート名とパスの関連を表示することもできます:

    .. code-block:: terminal

        $ symfony console debug:router

認可アクセスコントロールのルールを追加する
---------------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

セキュリティシステムは2つのパートによって構成されています。 *認証* と *認可* です。管理者ユーザーを作成した際に ``ROLE_ADMIN`` ロールを与えています。 ``access_control`` にルールを追加して、 ``ROLE_ADMIN`` ロールを持ったユーザーのみが ``/admin`` セクションにアクセスできるようにしましょう:

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

``access_control`` のルールは正規表現でアクセスを制限します。 ``/admin`` から始まる URL にアクセスされると、セキュリティシステムは、ログインしているユーザーが ``ROLE_ADMIN`` ロールがあるかチェックします。

ログインフォームで認証する
---------------------------------------

これで、管理者のバックエンドへのアクセスを試みると、ログインページにリダイレクトされ、ログインとパスワードの入力を促されるはずです:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

``admin`` と先ほど選んだパスワードを使ってログインしてください。 そのまま SQL をコピーしていたなら、そのパスワードの値は ``admin`` です。

EasyAdmin は自動的に Symfony の認証システムを検知します:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

"ログアウト" リンクをクリックしてください。これで、管理者のバックエンドはセキュアな状態になります。

.. index::
    single: Command;make:registration-form

.. note::

    ``make:registration:form`` コマンドを使えば、より高機能な認証システムを作成することができます。

.. sidebar:: より深く学ぶために

    * `Symfony セキュリティのドキュメント`_;

    * `SymfonyCasts セキュリティチュートリアル`_;

    * Symfony アプリケーションでの `ログインフォームの作り方`_;

    * `Symfony セキュリティのチートシート`_.

.. _`Symfony セキュリティのドキュメント`: https://symfony.com/doc/current/security.html
.. _`SymfonyCasts セキュリティチュートリアル`: https://symfonycasts.com/screencast/symfony-security
.. _`ログインフォームの作り方`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`Symfony セキュリティのチートシート`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
