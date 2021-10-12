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

.. code-block:: bash

    $ symfony composer req api

カンファレンス用のAPIを公開する
---------------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;Groups

Conferenceクラスにいくつかのアノテーションを設定するだけで、APIの設定を行うことができます。

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,16 +2,25 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiResource;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
      * @UniqueEntity("slug")
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="conference:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="conference:item"}}},
    + *     order={"year"="DESC", "city"="ASC"},
    + *     paginationEnabled=false
    + * )
      */
     class Conference
     {
    @@ -20,21 +29,25 @@ class Conference
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $city;

         /**
          * @ORM\Column(type="string", length=4)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $year;

         /**
          * @ORM\Column(type="boolean")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $isInternational;

         /**
    @@ -45,6 +58,7 @@ class Conference
         /**
          * @ORM\Column(type="string", length=255, unique=true)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $slug;

         public function __construct()

メインの ``@ApiResource`` アノテーションは、カンファレンス用のAPIを構成します。 可能な操作を ``get`` に制限して、表示するフィールドや表示順など様々な設定をしています。

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
    single: Annotations;@ApiResource
    single: Annotations;@ApiFilter
    single: Annotations;Groups

コメントにも同様の設定を行います。

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,13 +2,26 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiFilter;
    +use ApiPlatform\Core\Annotation\ApiResource;
    +use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
      * @ORM\HasLifecycleCallbacks()
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="comment:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="comment:item"}}},
    + *     order={"createdAt"="DESC"},
    + *     paginationEnabled=false
    + * )
    + *
    + * @ApiFilter(SearchFilter::class, properties={"conference": "exact"})
      */
     class Comment
     {
    @@ -17,18 +30,21 @@ class Comment
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $author;

         /**
          * @ORM\Column(type="text")
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $text;

         /**
    @@ -36,22 +52,26 @@ class Comment
          */
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $email;

         /**
          * @ORM\Column(type="datetime")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $createdAt;

         /**
          * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
          * @ORM\JoinColumn(nullable=false)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $conference;

         /**
          * @ORM\Column(type="string", length=255, nullable=true)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $photoFilename;

         /**

カンファレンスクラスで設定したのと同じ種類のアノテーションをクラスに設定します。

APIによって公開されるコメントを制限する
---------------------------------------------------------

デフォルトでは、API Platformはデータベースから全てのデータを出力します。ただし、コメントについては、公開が許可されたもののみが出力される必要があります。

APIによって返されるアイテムを制限する必要がある場合、コレクションの Doctrine クエリーを制御する ``QueryCollectionExtensionInterface`` や、アイテムを制御する ``QueryItemExtensionInterface`` を実装したサービスを作成します。

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 13-15,20-22

    namespace App\Api;

    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
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

    * `SymfonyCasts API Platform チュートリアル <https://symfonycasts.com/screencast/api-platform>`_;

    * GraphQLサポートを有効にするには、 ``composer require webonyx/graphql-php`` を実行し、 ``/api/graphql`` にアクセスします。
