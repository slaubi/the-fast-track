API Platformを使ってAPIを公開する
==========================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

ゲストブックのWebサイト実装が完了しました。より多くデータを利用してもらうために、APIを公開するのはどうでしょうか？APIをモバイルアプリケーションで利用して、カンファレンスやコメントを表示したり、参加者がコメントを残すことができるようにすることが可能です。

このステップでは、読み取り専用のAPIを実装します。

API Platform をインストールする
----------------------------------------

いくつかのコードを実装することでAPIを公開することはできますが、スタンダードなやり方を求める場合は、API Platformのようなすでに面倒な実装を行っているソリューションを使用した方が良いでしょう。

.. code-block:: terminal

    $ symfony composer req api

カンファレンス用のAPIを公開する
---------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Conferenceクラスにいくつかのアトリビュートを設定するだけで、APIの設定を行うことができます。

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,35 +2,52 @@

     namespace App\Entity;

    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    +#[ApiResource(
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'conference:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'conference:list'])
    +    ],
    +    order: ['year' => 'DESC', 'city' => 'ASC'],
    +    paginationEnabled: false,
    +)]
     class Conference
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $city = null;

         #[ORM\Column(length: 4)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $year = null;

         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?bool $isInternational = null;

         #[ORM\OneToMany(mappedBy: 'conference', targetEntity: Comment::class, orphanRemoval: true)]
         private Collection $comments;

         #[ORM\Column(length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $slug = null;

         public function __construct()

メインの ``ApiResource`` アトリビュートは、カンファレンス用のAPIを構成します。 可能な操作を ``get`` に制限して、表示するフィールドや表示順など様々な設定をしています。

デフォルトでは、APIのメインエントリーポイントは ``/api`` です。これは、API Platformのパッケージによって追加された ``config/routes/api_platform.yaml`` によって構成されています。

Webインターフェースを使用することで、APIと対話することができます。

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

これを使用して様々な可能性を検証します。

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

ここまでの全てをゼロから実装するためにかかる時間を想像してください！

コメントAPIの公開
------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

コメントにも同様の設定を行います。

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,41 +2,63 @@

     namespace App\Entity;

    +use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
    +use ApiPlatform\Metadata\ApiFilter;
    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\CommentRepository;
     use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    +#[ApiResource(
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'comment:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'comment:list'])
    +    ],
    +    order: ['createdAt' => 'DESC'],
    +    paginationEnabled: false,
    +)]
    +#[ApiFilter(SearchFilter::class, properties: ['conference' => 'exact'])]
     class Comment
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $author = null;

         #[ORM\Column(type: Types::TEXT)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $text = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $email = null;

         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?\DateTimeImmutable $createdAt = null;

         #[ORM\ManyToOne(inversedBy: 'comments')]
         #[ORM\JoinColumn(nullable: false)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?Conference $conference = null;

         #[ORM\Column(length: 255, nullable: true)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $photoFilename = null;

         #[ORM\Column(length: 255, options: ['default' => 'submitted'])]

カンファレンスクラスで設定したのと同じ種類のアトリビュートをクラスに設定します。

APIによって公開されるコメントを制限する
---------------------------------------------------------

デフォルトでは、API Platformはデータベースから全てのデータを出力します。ただし、コメントについては、公開が許可されたもののみが出力される必要があります。

APIによって返されるアイテムを制限する必要がある場合、コレクションの Doctrine クエリーを制御する ``QueryCollectionExtensionInterface`` や、アイテムを制御する ``QueryItemExtensionInterface`` を実装したサービスを作成します。

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 14-16,21-23

    namespace App\Api;

    use ApiPlatform\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use ApiPlatform\Metadata\Operation;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }
    }

このクエリ拡張クラスは ``Comment`` のリソースに対してのみロジックを適用し、Doctrineクエリービルダーを変更して、 ``published`` 状態のコメントのみ扱うようにします。

CORS を設定する
--------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

デフォルトでは、最新HTTPクライアントの同一生成元ポリシーにより、別ドメインからのAPI呼び出しは禁止されています。 ``composer req api`` によりインストールされるCORSバンドルは、環境変数 ``CORS_ALLOW_ORIGIN`` をもとに、オリジン間リソース共有ヘッダーを送信します。

デフォルトでは、 ``.env`` で定義されている ``localhost`` と ``127.0.0.1`` の任意ポートからのHTTPリクエストが許可されています。APIを呼び出す独自のWebサーバを作成するためには、次のステップが必要です。

.. sidebar:: より深く学ぶために

    * `SymfonyCasts API Platform チュートリアル`_;

    * GraphQLサポートを有効にするには、 ``composer require webonyx/graphql-php`` を実行し、 ``/api/graphql`` にアクセスします。

.. _`SymfonyCasts API Platform チュートリアル`: https://symfonycasts.com/screencast/api-platform
