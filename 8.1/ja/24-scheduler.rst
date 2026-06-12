Cron を実行する
====================

.. index::
    single: Cron

Cron は、メンテナンスタスクに便利です。ワーカーと異なり、短い期間でスケジュール実行されます。

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
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand extends Command
    {
        public function __construct(
            private CommentRepository $commentRepository,
        ) {
            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->addOption('dry-run', null, InputOption::VALUE_NONE, 'Dry run')
            ;
        }

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $io = new SymfonyStyle($input, $output);

            if ($input->getOption('dry-run')) {
                $io->note('Dry mode enabled');

                $count = $this->commentRepository->countOldRejected();
            } else {
                $count = $this->commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Symfony にビルトインされている全てのアプリケーションのコマンドは、 ``symfony console`` からアクセス可能です。使用可能なコマンドの数は、とても多くなるので、ネームスペースを付けてください。規約として、アプリケーションコマンドは、 ``app`` ネームスペース以下に格納してください。そして、コロン(``:``) で区切りを付けて、サブネームスペースを付けてください。サブネームスペースは1つでも複数個でも良いです。

コマンドは *input* （コマンドから渡ってきた引数やオプション）を取得し、 コンソールに書き出すのに *output* を使うことができます。

このコマンドを実行してデータベースをクリーンアップしてください:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Upsun で Cron をセットアップする
------------------------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun の便利な点の一つは、ほとんどの設定は ``.upsun/config.yaml`` に格納されていることです。メンテナンスを楽にするために、Webコンテナ、ワーカー、Cron ジョブが一緒に記述されています:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -83,5 +83,13 @@ applications:
                     spec: '17,47 * * * *'
                     commands:
                         start: croncape php-session-clean
    +            comment_cleanup:
    +                # Cleanup every night at 11.50 pm (UTC).
    +                spec: '50 23 * * *'
    +                commands:
    +                    start: |
    +                        if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
    +                            croncape symfony console app:comment:cleanup
    +                        fi

             workers:

``crons`` セクションは、全ての Cron ジョブを定義しています。各 Cron は、 ``spec`` スケジュールに応じて実行されます。

``croncap`` ユーティリティは、コマンドの実行をモニターして、コマンドが ``0`` でないコードで終わったときに、環境変数 ``MAILTO`` に定義されているメールアドレスにメールを送ります。

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

``MAILTO`` 環境変数を設定する

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Cron は、全ての Upsun のブランチにセットアップされています。本番以外の環境で Cron を走らせたくない場合は、環境変数の ``$PLATFORM_ENVIRONMENT_TYPE``  をチェックしてください:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: より深く学ぶために

    * `Cron/crontab のシンタックス`_;

    * `Croncape のリポジトリ`_;

    * `Symfony コンソールコマンド`_;

    * `Symfony コンソールのチートシート`_.

.. _`Cron/crontab のシンタックス`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape のリポジトリ`: https://github.com/symfonycorp/croncape
.. _`Symfony コンソールコマンド`: https://symfony.com/doc/current/console.html
.. _`Symfony コンソールのチートシート`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
