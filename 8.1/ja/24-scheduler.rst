タスクをスケジュールする
========================

.. index::
    single: Cron

メンテナンスタスクの中には、スケジュールに沿って実行しなければならないものがあります。継続的に実行されるワーカーと異なり、スケジュールされたタスクは短い時間、定期的に実行されます。

コメントをクリーンアップする
------------------------------------------

スパムとして判定されたコメント、管理者によって拒否されたコメントは、データベース内に残っているので、管理者は少しの間調べることがあるかもしれません。しかし、ある程度時間が経てば、これらのコメントは消されるべきです。コメントが作成されてから1週間くらい残しておくのがちょうど良いでしょう。

コメントリポジトリに、拒否されたコメントを取得したり、数を数えたり、削除したりするためのユーティリティメソッドを追加しましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -5,7 +5,9 @@ namespace App\Repository;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    +use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -13,6 +15,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const COMMENTS_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -20,6 +24,27 @@ class CommentRepository extends ServiceEntityRepository
             parent::__construct($registry, Comment::class);
         }

    +    public function countOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->select('COUNT(c.id)')->getQuery()->getSingleScalarResult();
    +    }
    +
    +    public function deleteOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->delete()->getQuery()->execute();
    +    }
    +
    +    private function getOldRejectedQueryBuilder(): QueryBuilder
    +    {
    +        return $this->createQueryBuilder('c')
    +            ->andWhere('c.state = :state_rejected or c.state = :state_spam')
    +            ->andWhere('c.createdAt < :date')
    +            ->setParameter('state_rejected', 'rejected')
    +            ->setParameter('state_spam', 'spam')
    +            ->setParameter('date', new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'))
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    より複雑なクエリーが必要な際は、生成されたSQLステートメントを確認すると便利です（これらはログや、Webリクエストのプロファイラで見つけることができます）。

クラス定数、コンテナのパラメーター、環境変数を使用する
---------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7日としましたが、10日にするかもしれないですし、20日にするかもしれません。この数値は時間と共に変更される可能性があります。ここでは、クラスの定数として格納すると決めましたが、コンテナのパラメータとして格納するかもしれませんし、環境変数として定義するかもしれません。

どのアブストラクションを使用するか決める経験則は以下の通りです:

* 値が注意が必要なもの（パスワードや APIトークン）だった際は、 Symfony の *シークレットストレージ* かヴォールトを使用してください;

* 値が動的に変わり、*デプロイすることなく* 変更したいときは、 *環境変数* を使用してください;

* 値が環境によって異なっていれば、 *コンテナパラメーター* を使用してください;

* その他のケースでは、 *クラス定数* のようにコードに値を格納してください。

CLI コマンドを作成する
-------------------------------

古くなったコメントを削除するのは、Cron ジョブの良いタスクです。そして、定期的に実行されるべきで、少し遅延しても大きな影響はありません。

``src/Command/CommentCleanupCommand.php`` ファイルを作成して、``app:comment:cleanup`` と命名した CLI コマンドを作成します:

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Attribute\Option;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand
    {
        public function __invoke(
            SymfonyStyle $io,
            CommentRepository $commentRepository,
            #[Option(description: 'Dry run')]
            bool $dryRun = false,
        ): int {
            if ($dryRun) {
                $io->note('Dry mode enabled');

                $count = $commentRepository->countOldRejected();
            } else {
                $count = $commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Symfony にビルトインされている全てのアプリケーションのコマンドは、 ``symfony console`` からアクセス可能です。使用可能なコマンドの数は、とても多くなるので、ネームスペースを付けてください。規約として、アプリケーションコマンドは、 ``app`` ネームスペース以下に格納してください。そして、コロン(``:``) で区切りを付けて、サブネームスペースを付けてください。サブネームスペースは1つでも複数個でも良いです。

コマンドは、 ``__invoke()`` のパラメータに付けた ``#[Argument]`` と ``#[Option]`` アトリビュートで *引数* と *オプション* を宣言します（ ``$dryRun`` パラメータは ``--dry-run`` オプションになります）。Symfony は他のパラメータをその型に基づいてインジェクトします。コンソールに綺麗にフォーマットされた出力を書き出すための ``SymfonyStyle`` や、コメントリポジトリのような任意のサービスを、コントローラーの引数と同じようにインジェクトします。

このコマンドを実行してデータベースをクリーンアップしてください:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

コマンドをスケジュールする
--------------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

コマンドを手動で実行することはできますが、毎晩実行されるべきです。Symfony Scheduler コンポーネントは、スケジュールに沿ってメッセージを生成します。生成されたメッセージは、他の Messenger メッセージと同じように、ワーカーによってコンシュームされます。

Scheduler コンポーネントを、cron 式をパースするライブラリと一緒に追加してください:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

``#[AsCronTask]`` アトリビュートでコマンドをスケジュールします:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/CommentCleanupCommand.php
    +++ w/src/Command/CommentCleanupCommand.php
    @@ -7,8 +7,10 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Attribute\Option;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Style\SymfonyStyle;
    +use Symfony\Component\Scheduler\Attribute\AsCronTask;

     #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    +#[AsCronTask('50 23 * * *')]
     class CommentCleanupCommand
     {
         public function __invoke(

このアトリビュートは、cron 式でデフォルトの *スケジュール* にコマンドを登録します。毎晩 23 時 50 分（UTC）です。確認してみましょう:

.. code-block:: terminal

    $ symfony console debug:scheduler

スケジュールは、その名前を持つ通常の Messenger トランスポートとして公開されます。他のトランスポートと同じようにコンシュームしてください:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

スケジュールをデプロイする
--------------------------

.. index::
    single: Upsun;Workers

Upsun では、ワーカーは ``async`` トランスポートだけをコンシュームしています。スケジュールもコンシュームするようにしましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -87,4 +87,4 @@ applications:
             messenger:
                 commands:
                     # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
    -                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async
    +                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async scheduler_default

必要なのはこれだけです。crontab も、追加のプロセスも要りません。スケジュールは、トリガーするタスクのすぐ隣の PHP コードの中にあり、アプリケーションの他の部分と同じようにデプロイされ、バージョン管理されます。

システム Cron はどうするのか？
------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun は、OS レベルの cron ジョブもサポートしています。 ``.upsun/config.yaml`` の中で、Webコンテナやワーカーと並んで記述されます。デフォルトの設定では、期限切れの PHP セッションをクリーンアップする cron ジョブが既に定義されています。システム cron は、PHP で実装されていないタスクに適しています。

デフォルトの cron で使用されている ``croncape`` ユーティリティは、コマンドの実行を監視し、コマンドが ``0`` 以外の終了コードを返した場合、 ``MAILTO`` 環境変数に定義されたアドレスへメールを送信します:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Cron は全ての Upsun ブランチでセットアップされることに注意してください。本番環境以外で実行したくない場合は、 ``$PLATFORM_ENVIRONMENT_TYPE`` 環境変数をチェックしてください:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: より深く学ぶために

    * `Scheduler コンポーネントのドキュメント`_;

    * `Cron/crontab のシンタックス`_;

    * `Croncape のリポジトリ`_;

    * `Symfony コンソールコマンド`_;

    * `Symfony コンソールのチートシート`_.

.. _`Scheduler コンポーネントのドキュメント`: https://symfony.com/doc/current/scheduler.html
.. _`Cron/crontab のシンタックス`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape のリポジトリ`: https://github.com/symfonycorp/croncape
.. _`Symfony コンソールコマンド`: https://symfony.com/doc/current/console.html
.. _`Symfony コンソールのチートシート`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
