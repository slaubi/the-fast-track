データ構造の説明
========================

.. index::
    single: Doctrine
    single: Database

PHP からデータベースを扱うのに、データベース運用に便利なライブラリの `Doctrine`_ を使用しましょう: Doctrine DBAL (データベース抽象化レイヤ)、 Doctrine ORM (PHPのオブジェクトを使ってデータベース上のデータを更新するライブラリ)、 そしてDoctrine Migrationsがあります。

Doctrine ORM の設定
----------------------

.. index::
    single: Doctrine;Configuration

Doctrine はどのようにデータベース接続に関して知るのでしょうか？Doctrine のレシピは ``config/packages/doctrine.yml`` という設定ファイルに必要な情報を書き込んでいます。主な設定項目は、クレデンシャルや host, port などの接続に関する情報を含んだ文字列である *データベースのDSN* です。デフォルトでは、 Doctrine は、 ``DATABASE_URL`` 環境変数を探すようになっています。

ほとんど全てのインストール済みパッケージは、 ``config/package/`` ディレクトリ内に設定ファイルがあります。大抵の場合、デフォルト値は、ほとんどのアプリケーションで動作するよう、慎重に設定されています。

Symfony の環境変数の規約を理解する
-----------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``.env`` や ``.env.local`` ファイルに手動で ``DATABASE_URL`` を定義することもできます。実際、パッケージのレシピを使うことで、 ``.env`` ファイル内に ``DATABASE_URL`` がサンプルが書かれていますので確認してください。しかし、Docker が PostgreSQL のローカルポートは変えることがありますので、少し面倒なときもあります。そんなときのために、より良い方法があります。

ファイル内に ``DATABASE_URL`` をハードコードするのではなく、全てのコマンドに ``symfony`` の接頭辞をつけることができます。このことによって Docker や Platform.sh (トンネルが開いていれば)で動いているサービスを検知し、環境変数を自動的に設定してくれます。

これらの環境変数を使うことで Docker Compose と Platform.sh は、Symfony とシームレスに連携することができます。

.. index::
    single: Symfony CLI;var:export

``symfony var:export`` を実行して、全ての公開されている環境変数をチェックしてください:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Docker や Platform.sh の設定で使われている *サービス名* ``database`` を覚えていますか？サービス名は、``DATABASE_URL`` のような環境変数を定義の接頭辞として使われます。Symfony の規約に沿ってサービスが命名されていれば、特に設定は必要ありません。

.. note::

    Symfony の規約による利点は、データベースのサービスのみではなく、メーラーなどにもあります(環境変数 ``MAILER_DSN`` )

.env のデフォルトの DATABSE_URL の値を変更する
------------------------------------------------------------

``.env`` ファイルを変更して、PostgreSQL を使うためのデフォルトの ``DATABASE_URL`` をセットアップしましょう:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

なぜ別の場所に重複した情報を設定する必要があるのでしょうか？クラウドによっては *ビルド時* にデータベースの URL がわからないものもあり、Doctrine が設定情報のために、データベースエンジンを知る必要があるからです。ホスト名やユーザー名、パスワードは知る必要はありません。

エンティティクラスを作成する
------------------------------------------

conference オブジェクトはプロパティで記述することができます:

* *city* はカンファレンスの開催場所です;

* *year* は、カンファレンスの開催年です;

* *international* フラグは、カンファレンスがローカルイベントなのか国際イベントなのかを表します（SymfonyLive vs SymfonyCon）。

.. index:: ! Command;make:entity

Maker バンドルで conference を表現するための *エンティティ* クラスを生成することができます。

さあ ``Conference`` エンティティを作りましょう:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

このコマンドはインタラクティブに動きます: 全ての必要なフィールドを追加することを補助してくれます。以下のように答えましょう（ほとんどはデフォルトで、ただ "エンター"キーを押すだけで良いはずです）:

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

コマンドの実行結果の全出力は以下になります:

.. code-block:: text
    :emphasize-lines: 8,11,14,17,22,25,28,31,36,39,42,45
    :class: ignore

     created: src/Entity/Conference.php
     created: src/Repository/ConferenceRepository.php

     Entity generated! Now let's add some fields!
     You can always add more fields later manually or by re-running this command.

     New property name (press <return> to stop adding fields):
     > city

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > year

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     > 4

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > isInternational

     Field type (enter ? to see all types) [boolean]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     >



      Success!


     Next: When you're ready, create a migration with make:migration

``Conference`` クラスは、 ``App\Entity\`` ネームスペース以下に作成されます。

このコマンドは、 Doctrine の *リポジトリ* クラスも ``App\Repository\ConferenceRepository`` に生成してくれます 。

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

生成されたコードは次のようになっています（ここではファイルを少しだけ表示しています）:

.. code-block:: php
    :caption: src/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    class Conference
    {
        #[ORM\Column(type: 'integer')]
        #[ORM\Id, ORM\GeneratedValue()]
        private $id;

        #[ORM\Column(type: 'string', length: 255)]
        private $city;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): self
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

クラス自体は、Doctrine に関する情報がない、ただの PHP クラスです。Doctrineのアトリビュートを使うことで、関連するデータベースのテーブルとのマッピングをすることができます。

Doctrine は、データベースのテーブルのプライマリーキーとして使用する ``id`` プロパティを追加します。 このキー(``ORM\Id()``) は、データベースエンジンによって自動的に生成されます(``ORM\GeneratedValue()``)。

.. index::
    single: Command;make:entity

カンファレンスのコメントのエンティティクラスを生成しましょう:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

次の回答を入力してください:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

エンティティ間の関連付け
------------------------------------

.. index::
    single: Command;make:entity

カンファレンスとコメントのエンティティは連携する必要があります。カンファンレスは n個のコメントを持ちますので、*one-to-many* の関連となります。

``Conference`` クラスにリレーションを追加するために、``make:entity`` コマンドをもう一度使いましょう:

.. code-block:: terminal
    :class: answers(comments||OneToMany||Comment||conference||no||yes)

    $ symfony console make:entity Conference

.. code-block:: text
    :emphasize-lines: 4,7,10,15,18,27
    :class: ignore

     Your entity already exists! So let's add some new fields!

     New property name (press <return> to stop adding fields):
     > comments

     Field type (enter ? to see all types) [string]:
     > OneToMany

     What class should this entity be related to?:
     > Comment

     A new property will also be added to the Comment class...

     New field name inside Comment [conference]:
     >

     Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
     > no

     Do you want to activate orphanRemoval on your relationship?
     A Comment is "orphaned" when it is removed from its related Conference.
     e.g. $conference->removeComment($comment)

     NOTE: If a Comment may *change* from one Conference to another, answer "no".

     Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
     > yes

     updated: src/Entity/Conference.php
     updated: src/Entity/Comment.php

.. note::

    タイプの回答に ``?`` を入力すると、全てのサポートしているタイプを表示することができます:

    .. code-block:: text
        :class: ignore

        Main types
          * string
          * text
          * boolean
          * integer (or smallint, bigint)
          * float

        Relationships / Associations
          * relation (a wizard will help you build the relation)
          * ManyToOne
          * OneToMany
          * ManyToMany
          * OneToOne

        Array/Object Types
          * array (or simple_array)
          * json
          * object
          * binary
          * blob

        Date/Time Types
          * datetime (or datetime_immutable)
          * datetimetz (or datetimetz_immutable)
          * date (or date_immutable)
          * time (or time_immutable)
          * dateinterval

        Other Types
          * decimal
          * guid
          * json_array

.. index::
    single: Attributes;ORM\\ManyToOne
    single: Attributes;ORM\\JoinColumn
    single: Attributes;ORM\\OneToMany

リレーションを追加した前後のエンティティクラスの diff を見てみましょう:

.. code-block:: diff
    :class: ignore

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -36,6 +36,12 @@ class Comment
          */
         private $createdAt;

    +    #[ORM\ManyToOne(inversedBy: 'comments')]
    +    #[ORM\JoinColumn(nullable: false)]
    +    private Conference $conference;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +94,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): self
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -31,6 +33,16 @@ class Conference
          */
         private $isInternational;

    +    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: "conference", orphanRemoval: true)]
    +    private $comments;
    +
    +    public function __construct()
    +    {
    +        $this->comments = new ArrayCollection();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -71,4 +83,35 @@ class Conference

             return $this;
         }
    +
    +    /**
    +     * @return Collection<int, Comment>
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): self
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments[] = $comment;
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): self
    +    {
    +        if ($this->comments->contains($comment)) {
    +            $this->comments->removeElement($comment);
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

リレーションに必要な全ては生成されました。生成されたコードは自由に編集してカスタマイズできます。

さらにプロパティを追加する
---------------------------------------

.. index::
    single: Command;make:entity

コメントエンティティにもう一つプロパティを追加するのを忘れていました: 参加者はフィードバックとしてカンファレンスの写真を添付するかもしれません。

``make:entity`` をもう一度実行し、``photoFilename`` プロパティ/カラムを追加しましょう。``string`` タイプですが、写真はオプショナルですので、 ``null`` を許容できるようにしましょう:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

データベースのマイグレーション
---------------------------------------------

.. index:: ! Command;make:migration

これで、２つの生成されたクラスでプロジェクトのモデルを記述できました。

次に、これらの PHP のエンティティに関連するデータベースのテーブルを作成する必要があります。

そのニーズには、*Doctrine Migrations* が使えます。 ``orm`` をインストールした際に一緒に入っています。

*マイグレーション* は、データベーススキーマをエンティティアトリビュートで定義した新しい状態に更新するのに必要な変更を記述するクラスです。まだデータベースに何も作成していないので、マイグレーションは、テーブルを2つ作成することになります。

Doctrine の生成物を見てみましょう:

.. code-block:: terminal

    $ symfony console make:migration

生成されたファイル名が出力されます(``migrations/Version20191019083640.php`` のような名前のファイル):

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version00000000000000 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema): void
        {
            // ...
        }
    }

ローカルデータベースの更新
---------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

これで、生成されたマイグレーションを走らせてローカルのデータベーススキーマを更新することができます

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

これでローカルのデータベーススキーマは最新になりましたので、値を入れる準備ができました。

本番のデータベースを更新する
------------------------------------------

本番のデータベースのマイグレーションに必要なステップは、既にやった内容と同じです: コミットして変更をデプロイするのみです。

プロジェクトをデプロイすると、 Platform.sh はコードを更新し、必要であればデータベースマイグレーションを実行します(``doctrine:migrations:migrate`` コマンドがあるかを検知します)。

.. sidebar:: より深く学ぶために

    * `Databases と Doctrine ORM`_ Symfony アプリケーション;

    * `SymfonyCasts Doctrine チュートリアル`_;

    * `Doctrine Associations/Relations を使用する`_;

    * `DoctrineMigrationsBundle のドキュメント`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Databases と Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`SymfonyCasts Doctrine チュートリアル`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Doctrine Associations/Relations を使用する`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle のドキュメント`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
