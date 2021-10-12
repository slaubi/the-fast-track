管理者用のバックエンドをセットアップする
============================================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

データベースに次のカンファレンスを追加するのは、プロジェクトの管理者の仕事です。 *管理者用のバックエンド* は、Webサイトの保護された場所となり、そこで *プロジェクト管理者* は、Webサイトのデータを管理したり、フィードバックをモデレートしたりできます。

早く作成するのにどうしましょうか？プロジェクトのモデルに基づく管理者用のバックエンドを生成することができる EasyAdmin バンドルを使いましょう。

EasyAdmin を設定する
-------------------------

まず、EasyAdmin をプロジェクトの依存に追加します:

.. code-block:: bash

    $ symfony composer req "admin:^2"

EasyAdmin を設定するのに、Flexレシピで新しい設定ファイルを生成します:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

このように、ほとんどのインストールされたパッケージは、 ``config/packages`` ディレクトリに設定を1つ持っています。ほとんどのアプリケーションで動作するように、デフォルトが設定されています。

最初のいくつかのコメントアウト行を取って、プロジェクトのモデルクラスを追加しましょう:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

生成された ``admin`` の管理者用バックエンドへアクセスしてみてください。Boom! カンファレンスとコメントの素敵でリッチなインターフェースができました:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

.. tip::

    Why is the backend accessible under ``/admin``? That's the default prefix configured in ``config/routes/easy_admin.yaml``:

    .. code-block:: yaml
        :caption: config/routes/easy_admin.yaml
        :class: ignore

        easy_admin_bundle:
            resource: '@EasyAdminBundle/Controller/EasyAdminController.php'
            prefix: /admin
            type: annotation

    You can change it to anything you like.

Adding conferences and comments is not possible yet as you would get an error: ``Object of class App\Entity\Conference could not be converted to string``. EasyAdmin tries to display the conference related to comments, but it can only do so if there is a string representation of a conference. Fix it by adding a ``__toString()`` method on the ``Conference`` class:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -44,6 +44,11 @@ class Conference
             $this->comments = new ArrayCollection();
         }

    +    public function __toString(): string
    +    {
    +        return $this->city.' '.$this->year;
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

`Comment` クラスと同じようにしましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -48,6 +48,11 @@ class Comment
          */
         private $photoFilename;

    +    public function __toString(): string
    +    {
    +        return (string) $this->getEmail();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

これで、管理者用バックエンドからカンファレンスを直接追加/変更/削除することができるようになりました。一つ以上カンファレンスを作成して遊んでみましょう。

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

写真なしのコメントを追加しましょう。ここでは日付は手動でセットしましょう; 後のステップで ``createdAt`` カラムを自動的にセットするようにします。

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

EasyAdmin をカスタマイズする
-------------------------------------

デフォルトの管理者用のバックエンドが正しく動くようになりましたが、カスタマイズしてさらに改善することができます。簡単な修正をしてどんなことができるか見てみましょう。現在の設定を次のように変更してみましょう:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        site_name: Conference Guestbook

        design:
            menu:
                - { route: 'homepage', label: 'Back to the website', icon: 'home' }
                - { entity: 'Conference', label: 'Conferences', icon: 'map-marker' }
                - { entity: 'Comment', label: 'Comments', icon: 'comments' }

        entities:
            Conference:
                class: App\Entity\Conference

            Comment:
                class: App\Entity\Comment
                list:
                    fields:
                        - author
                        - { property: 'email', type: 'email' }
                        - { property: 'createdAt', type: 'datetime' }
                    sort: ['createdAt', 'ASC']
                    filters: ['conference']
                edit:
                    fields:
                        - { property: 'conference' }
                        - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                        - 'author'
                        - { property: 'email', type: 'email' }
                        - text

We have overridden the ``design`` section to add icons to the menu items and to add a link back to the website home page.

For the ``Comment`` section, listing the fields lets us order them the way we want. Some fields are tweaked, like setting the creation date to read-only. The ``filters`` section defines which filters to expose on top of the regular search field.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

これらのカスタマイズで、EasyAdmin を使って可能なことを少し紹介をしました。

カンファレンスでコメントをフィルターしたり、メールアドレスでコメントを検索したりして、管理者画面を遊んでみてください。問題が一つあるとすれば、誰もがバックエンドにアクセスできるようになっていることですが、後のステップでセキュアにしていきますので、心配しないでください。

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: より深く学ぶために

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Symfony フレームワーク設定リファレンス <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
