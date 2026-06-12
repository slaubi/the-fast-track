Doctrine オブジェクトのライフサイクルを管理する
==================================================================

新しくコメントをした際には、自動的に現在の日時が ``createdAt`` としてセットされると良いですね。

Doctrine は、データベースに追加されるときや更新されるときといったライフサイクルにおいてオブジェクトやプロパティを操作するいろいろな方法があります。

ライフサイクルのコールバックを定義する
---------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

サービスの依存が必要なく、エンティティを1つしか操作しないときは、エンティティクラスにコールバックを定義すると良いでしょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -57,8 +57,6 @@ class CommentCrudController extends AbstractCrudController
             ]);
             if (Crud::PAGE_EDIT === $pageName) {
                 yield $createdAt->setFormTypeOption('disabled', true);
    -        } else {
    -            yield $createdAt;
             }
         }
     }
    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -86,6 +87,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue(): void
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

``ORM\PrePersist`` は、最初にデータベースに保存されたときにトリガーとして呼ばれる *イベント* です。このイベントの際に ``setCreatedAtValue()`` メソッドが呼ばれ、現在の日時が ``createdAt`` プロパティにセットされます。

カンファンレンスへスラッグを追加する
------------------------------------------------------

``/conference/1`` といったカンファレンスの URL は特に意味はありません。これはデータベースのプライマリーキーといった実装の詳細に依るものになっています。

代わりに ``/conference/paris-2020`` といった URL はどうですか？こちらの方が良いですね。 ``paris-2020`` はカンファレンスの *スラッグ* と呼んでいます。

.. index::
    single: Command;make:entity

カンファレンスに ``slug`` プロパティを追加しましょう （ 255文字の長さで nullable でない型です）:

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

新しいカラムを追加するのでマイグレーションファイルを作成しましょう:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

新しいマイグレーションを実行します:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

エラーになりましたが、想定内のことです。先程スラッグは ``null`` にならないように指定したのですが、マイグレーションを走らせると既存のカンファレンスのエンティティは ``null`` となってしまうからです。修正してみましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/migrations/Version00000000000000.php
    +++ w/migrations/Version00000000000000.php
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

ここでは、カラムを追加し、 ``null`` を許容した後に、スラッグに ``null`` でない値をセットします。最後に、スラッグのカラムを ``null`` 不可にしています。

.. note::

    実際のプロジェクトでは、 ``CONCAT(LOWER(city), '-', year)`` ではなく、 "本当の" スラッグを使用する必要があります。

.. index::
    single: Command;doctrine:migrations:migrate

これでマイグレーションが正しく動くはずです:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

これで各カンファレンスを探すためにスラッグを使うようにしたので、カンファレンスエンティティを修正して、スラッグがデータベース上でユニークになるようにしましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -6,8 +6,10 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    +#[UniqueEntity('slug')]
     class Conference
     {
         #[ORM\Id]
    @@ -30,7 +32,7 @@ class Conference
         #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

既にわかっているとは思いますが、ここでマイグレーションをする必要があります:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

スラッグを生成する
---------------------------

.. index::
    single: Components;String
    single: Slug

URL は、ASCII 文字以外を変換する必要があり、正しくスラッグを生成することは、英語圏以外の言語にとって難しいです。例えば、 ``é`` を ``e`` に変換する必要があります。

車輪の再発明をせずに Symfony の ``String`` コンポーネントを使いましょう。 文字列から *スラッグを生成* する方法が実装されています。

``Conference`` クラスに、カンファレンスの情報からスラッグを生成する ``computeSlug()`` メソッドを追加します:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -50,6 +51,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger): void
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

``computeSlug()`` メソッドは、現在のスラッグが何も指定していないか ``-`` と値が渡ったときのみ動作します。``-`` の値は、バックエンドでカンファレンスを追加するときにスラッグが必須となるので使用します。空ではないこの特別な値でアプリケーションにスラッグを自動生成させることができます。

複雑なライフサイクルのコールバックを定義する
------------------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

``createdAt`` プロパティのように ``slug`` も更新時に ``computeSlug()`` メソッドを呼べば自動的にセットされるようにした方が良いですね。

このメソッドは ``SluggerInterface`` の実装に依存していますので、以前のように ``prePersist`` イベントに追加することはできません。

代わりに Doctrineエンティティのリスナーを作成しましょう:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\PrePersistEventArgs;
    use Doctrine\ORM\Event\PreUpdateEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
        }

        public function prePersist(Conference $conference, PrePersistEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, PreUpdateEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }
    }

新しくカンファレンスが追加されたとき（``perPersist()``）と更新されたとき（``preUpdated()``）に、スラッグは更新されます。

コンテナにサービスを設定する
------------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

まだ、Symfony の鍵となるコンポーネント *DIコンテナ* について話していませんでした。このコンテナは、 *サービス* を作成したり必要なときにインジェクトしたりといった管理を行います:

*サービス* は "グローバル" なオブジェクトで、メーラーやロガーやスラッグ作成などの機能を提供します。これらは Doctrine のエンティティのインスタンスのような *データオブジェクト* とは違います。

実際は、必要なときにサービスが自動的にインジェクトされるのでコンテナを直接使うことはあまりありません。コンテナは型宣言によってコントローラの引数のオブジェクトを注入します。

前のステップでイベントリスナーがどうやって登録されたか不思議に思いませんでしたか？コンテナがその役割を担っていました。クラスが特定のインターフェースを実装すると、コンテナは、そのクラスがどうやって登録されるか知ることになるのです。

しかしここでは、クラスはインターフェースの実装や基底クラスの拡張をしていないので、Symfonyは自動的に設定することができません。代わりに、アトリビュートを使って、Symfonyコンテナに登録します:

.. code-block:: diff
    :caption: patch_file

    --- i/src/EntityListener/ConferenceEntityListener.php
    +++ w/src/EntityListener/ConferenceEntityListener.php
    @@ -3,10 +3,14 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\PrePersistEventArgs;
     use Doctrine\ORM\Event\PreUpdateEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    Doctrine のイベントリスナーとSymfony のイベントリスナーは同じように見えますが、内部では異なるインフラストラクチャーを使っており別物ですので注意してください。

アプリケーションでスラッグを使用する
------------------------------------------------------

バックエンドでさらにカンファレンスを追加したり、既に登録されているものの年や都市を変更してみましょう。 ``-`` を値として使用しなければ、スラッグは更新されません。

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

最後に行う変更として、コントローラーやテンプレートでカンファレンスの ``id`` の代わりに ``slug`` をルートで使用するように修正しましょう。ルートパラメータがエンティティのプライマリーキーではなくなったため、明示的な ``mapping`` を渡して、``#[MapEntity]`` にどのプロパティを対応させるかを指定します:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -20,7 +20,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    #[Route('/conference/{slug}', name: 'conference')]
    +    public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -16,7 +16,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
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

これでカンファレンスのページへスラッグから辿ることができるようになりました:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: より深く学ぶために

    * `Doctrine イベントシステム`_ (ライフサイクルコールバックとリスナーとエンティティリスナーとライフサイクルサブスクライバー);

    * `String コンポーネントのドキュメント`_;

    * `サービスコンテナ`_;

    * `Symfony サービスの Cheat Sheet`_.

.. _`Doctrine イベントシステム`: https://symfony.com/doc/current/doctrine/events.html
.. _`String コンポーネントのドキュメント`: https://symfony.com/doc/current/components/string.html
.. _`サービスコンテナ`: https://symfony.com/doc/current/service_container.html
.. _`Symfony サービスの Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
