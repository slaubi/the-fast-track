管理 Doctrine 对象的生命周期
=====================================

当新增一条评论时，``createdAt`` 这个字段最好可以自动设为当前的日期和时间。

Doctrine 在对象的生命周期中（在新行被插入到数据库之前、在对应行被更新之后......），有不同的方式来操作对象以及对象的属性。

定义生命周期的回调方法
---------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

当要执行的行为无需任何服务对象，而且只是针对一种实体类时，可以在该实体类里定义一个回调方法：

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\ORM\Mapping as ORM;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    + * @ORM\HasLifecycleCallbacks()
      */
     class Comment
     {
    @@ -106,6 +107,14 @@ class Comment
             return $this;
         }

    +    /**
    +     * @ORM\PrePersist
    +     */
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTime();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

对象的数据被第一次存储在数据库时会触发 ``@ORM\PrePersist`` *事件*。当触发时，``setCreatedAtValue()`` 方法会被调用，当前的日期和时间就会用来设置 ``createdAt`` 属性的值。

为会议添加 *slug*
----------------------

会议页面的 URL 对会议缺乏描述性：``/conference/1``。更重要的是，这些 URL 依赖于一个实现细节（数据库表的主键被泄露了）。

用类似 ``/conference/paris-2020`` 这样的 URL 来代替怎么样？那看上去好得多。``paris-2020`` 就是我们所谓的会议 *slug*。

.. index::
    single: Command;make:entity

为会议增加一个 ``slug`` 属性（一个不能取值 null 的 255 长度的字符串）：

.. code-block:: bash
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

添加一个数据库结构迁移文件，用来增加一个新的列：

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

执行这个新的结构迁移：

.. code-block:: bash
    :class: ignore

    $ symfony console doctrine:migrations:migrate

得到一个错误？这是预料之中的。为什么呢？因为我们要求这个 slug 不能是 null 值，但是当结构迁移执行时，数据库中已有的会议行会为 slug 设置一个 ``null`` 值。让我们通过调整一下该迁移文件来修复这个错误。

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema): void

这里的技巧是先增加这个 slug 列并且允许它取值为 ``null``，然后再把行里的 slug 设置成一个非 ``null`` 值，最后把这个列设置回不能取 ``null`` 值。

.. note::

    对于真实项目，在 SQL 中使用 ``CONCAT(LOWER(city), '-', year)`` 可能还不足以解决问题。在这种情况下，我们需要一个 *真正* 的 Slugger。

.. index::
    single: Command;doctrine:migrations:migrate

数据库结构迁移现在应该可以顺利执行了：

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

因为应用程序马上会使用 slug 来查找每个会议，我们来调整下会议的实体类，确保 slug 的值在数据库中是唯一的：

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,9 +6,11 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    + * @UniqueEntity("slug")
      */
     class Conference
     {
    @@ -40,7 +42,7 @@ class Conference
         private $comments;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, unique=true)
          */
         private $slug;

.. index::
    single: Components;Validator

因为我们使用了验证器来确保 slug 的唯一性，我们需要增加 Symfony 的 Validator 组件：

.. code-block:: bash

    $ symfony composer req validator

.. index::
    single: Command;make:migration

正如你所料，我们需要执行一次数据库结构迁移：

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

生成 slug
-----------

.. index::
    single: Components;String
    single: Slug

生成一个可读性良好的 slug，并将它用于 URL（任何非 ASCII 字符都会被编码），这是个有挑战的工作，尤其是对于英语之外的语言。比如你该如何把 ``é`` 转换成 ``e`` 呢？

不必重新发明轮子，让我们来用 Symfony 的 ``String`` 组件，它使字符串操作变得很容易，而且它也提供了一个 *slugger*：

.. code-block:: bash

    $ symfony composer req string

在 ``Conference`` 类里增加一个 ``computeSlug()`` 方法，它会根据会议数据计算出 slug 的值：

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    @@ -61,6 +62,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger)
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

只有当前 slug 为空或者是设为了 ``-`` 这个特殊值的时候，``computeSlug()`` 方法才会计算 slug 值。为什么我们要用 ``-`` 这个特殊值呢？因为在后台新增一个会议时，slug 是必填的。所以我们需要一个非空值，它告诉应用程序，我们要自动生成 slug。

定义一个复杂的生命周期回调方法
---------------------------------------------

.. index::
    single: Doctrine;Entity Listener

和 ``createdAt`` 一样，``slug`` 应该被自动设置。当会议被更新时，``computeSlug()`` 方法要被自动调用。

但是这个方法依赖于 ``SluggerInterface`` 接口的一个实现，所以我们无法像之前那样增加一个 ``prePersist`` 事件（我们无法注入 slugger）。

转而创建一个 Doctrine 里针对实体的监听器：

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        private $slugger;

        public function __construct(SluggerInterface $slugger)
        {
            $this->slugger = $slugger;
        }

        public function prePersist(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }
    }

注意到当新建或更新一个会议时，slug 也会更新（新建时调用 ``prePersist()`` 方法，更新时调用 ``preUpdate()`` 方法）。

在服务容器中配置服务
------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

到目前为止，我们还没有谈到 Symfony 中一个重要的组成部分，就是 *依赖注入容器*，它负责管理各个 *服务*：在需要的时候创建和注入服务。

一个 *服务* 是一个提供某项功能的“全局”对象（比如一个 *mailer* 、一个 *logger* 、一个 *slugger* 等等），而不是一个 *数据对象* （比如 Doctrine 实体的实例）。

你极少需要去直接操作容器，因为在你需要服务的时候，它会自动注入服务对象：例如，当你在控制器方法里对参数进行类型提示时，容器会注入对应类型的服务对象。

如果在前面的步骤里，你想要知道事件的监听器是如何注册的，那么现在你有答案了：那就是容器。当某个类实现了一个特定的接口，容器就会知道应该以某种方式注册这个类。

不过很可惜，自动化并非总是可以实现，尤其在一些第三方包中。我们刚才写的这个针对实体的监听器就是一个例子。由于它没有实现任何接口，也没有继承自一个容器的“已知类”，所以 Symfony 的服务容器并不能自动管理它。

我们需要在容器中部分地声明这个监听器。容器仍然可以猜测出依赖关联，所以不用把它写出来，但我们需要手工添加一些“标签”来把这个监听器注册到 Doctrine 的事件分发器：

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -29,3 +29,7 @@ services:

         # add more service definitions when explicit configuration is needed
         # please note that last definitions always *replace* previous ones
    +    App\EntityListener\ConferenceEntityListener:
    +        tags:
    +            - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
    +            - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}

.. note::

    不要把 Doctrine 的事件监听器和 Symfony 的事件监听器混淆起来。尽管它们看上去很相似，但它们用的底层代码架构是不一样的。

在应用中使用 slug
-----------------------

试着在后台里添加更多会议，并且更新一个现有会议的城市或年份；除非你使用特殊的 ``-`` 值，否则 slug 不会更新。

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;Route

最后一个改动，就是更新控制器和模板，让它们在路由中使用会议的 ``slug`` 来代替 ``id``。

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
             ]));
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    +    #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -18,7 +18,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -22,10 +22,10 @@
             {% endfor %}

             {% if previous >= 0 %}
    -            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
             {% endif %}
             {% if next < comments|length %}
    -            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
             {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>

现在会议页面应该可以使用它的 slug 来打开：

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: 深入学习

    * `Doctrine 的事件系统 <https://symfony.com/doc/current/doctrine/events.html>`_ （生命周期回调和监听器，实体的监听器和生命周期订阅器）；

    * `String 组件文档 <https://symfony.com/doc/current/components/string.html>`_；

    * `服务容器 <https://symfony.com/doc/current/service_container.html>`_；

    * `Symfony 服务速查表 <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_。
