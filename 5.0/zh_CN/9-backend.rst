设置管理后台
==================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

把将要举办的会议录入到数据库是项目管理员的工作。所谓的 *管理后台* 是网站中一个受保护的区域，用来让 *项目管理员* 管理网站数据，处理提交的反馈和做其它一些事。

我们如何快速做一个后台呢？通过用一个 bundle，它可以根据项目的数据模型来生成后台！EasyAdmin 最适合不过了。

配置 EasyAdmin
----------------

首先，把 EasyAdmin 加进项目的依赖中。

.. code-block:: bash

    $ symfony composer req "admin:^2"

它借助 Flex 的 recipe 生成了一个新的配置文件，可以用该文件来配置 EasyAdmin。

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

几乎所有安装包都有一个像这样的配置文件放在 ``config/packages/`` 目录下。大多数时候，精心选择的默认值在大部分项目里都能工作良好。

去掉前面几行的注释，并加上这个项目的模型类：

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

用 ``/admin`` 路径访问这个生成的后台。哇！有了一个漂亮且功能丰富的后台界面，可以管理会议和评论：

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

对 ``Comment`` 类也做同样的处理：

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

现在你可以在管理后台里直接添加/修改/删除会议。你可以玩玩看，并且添加至少一个会议。

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

添加几个不带照片的评论。现在先手工设置下日期；在后面的步骤中，我们会让 ``createdAt`` 列自动填充。

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

定制 EasyAdmin
----------------

默认的后台运行得很好，但是我们可以在许多方面对它进行定制，以提高使用体验。让我们对它做一些简单的改变来展现定制的可能性。用下面的内容代替当前的配置：

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

这些改动只是对于 EasyAdmin 定制可能性的一个小小介绍。

在后台里玩玩，比如可以用会议来过滤评论，或者根据邮件来搜索评论。目前唯一的问题是任何人都可以进入后台。别担心，我们会在以后的步骤中将它保护起来。

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: 深入学习

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Symfony 框架的配置参考 <https://symfony.com/doc/current/reference/configuration/framework.html>`_。
