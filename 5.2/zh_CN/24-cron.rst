运行 cron 定时任务
========================

.. index::
    single: Cron

cron 定时任务对维护工作很有用。与 worker 进程不同，它们是定时执行的，并且时间较短。

清理评论
------------

被标记为垃圾信息的评论，以及被管理员拒绝的评论，它们仍会保留在数据库，因为管理员可能会在一段时间内对它们进行检查。在过了一阵后，很可能需要把它们清理掉。这样的评论从创建后保留一星期应该够了。

在评论对应的 repository 类里新增一些工具方法，用来找到被拒绝的评论，清点这些评论的总数以及删除它们：

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -6,6 +6,7 @@ use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -16,6 +17,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -23,6 +26,29 @@ class CommentRepository extends ServiceEntityRepository
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
    +            ->setParameters([
    +                'state_rejected' => 'rejected',
    +                'state_spam' => 'spam',
    +                'date' => new \DateTime(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    对于更复杂的查询，有时候看一下生成的 SQL 语句会很有帮助（你能在日志和 web 请求分析器中找到这些语句）。

使用类常量，容器参数和环境变量
---------------------------------------------

.. index::
    single: Container;Parameters

7 天？我们本来也可以用另外一个数字，或许是 10 天或 20 天。这个数字可能会随着时间推移而改变。我们决定了把它存储为类里的一个常量，但我们也可以将它存储为容器里的一个参数，甚至可以把它定义为一个环境变量。

这里是一些决定使用哪一种抽象时的经验法则：

* 如果这个值是敏感信息（密码，API 令牌等），就用 Symfony 的 *机密存储* 或保险箱机制；

* 如果这个值是动态的，并且你需要 *不* 经重新部署就能去更新它，那么就使用 *环境变量*；

* 如果这个值可能依据环境而不同，那么就用 *容器参数*；

* 对于其它情况，就把这个值存储在代码中，比如在一个 *类常量* 中。

创建一个控制台命令
---------------------------

用 cron 任务来删除旧评论非常合适。它应该定期执行，而且稍有耽误也不会有大的影响。

通过新建 ``src/Command/CommentCleanupCommand.php`` 文件来创建一个名为 ``app:comment:cleanup`` 的新命令：

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    class CommentCleanupCommand extends Command
    {
        private $commentRepository;

        protected static $defaultName = 'app:comment:cleanup';

        public function __construct(CommentRepository $commentRepository)
        {
            $this->commentRepository = $commentRepository;

            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->setDescription('Deletes rejected and spam comments from the database')
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

            return 0;
        }
    }

应用程序的所有命令会和 Symfony 自带的命令注册在一起，它们都可以通过 ``symfony console`` 来执行。因为可用的命令数量很多，你应该用命名空间来管理它们。习惯上应用程序自己的命令会在 ``app`` 命名空间下。你可以用冒号（``:``）来增加任意数量的子命名空间。

一个命令接收 *输入* （传递给命令的参数和选项），你也可以把 *输出* 写到控制台上。

通过执行这个命令来清理数据库：

.. code-block:: bash

    $ symfony console app:comment:cleanup

在 SymfonyCloud 上设置 cron 定时任务
--------------------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

使用 SymfonyCloud 的好处之一就是大部分配置都存储在一个文件中：``.symfony.cloud.yaml``。web 容器，worker 进程还有 cron 任务的配置描述都放在一起，这样有利于维护：

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -52,6 +52,15 @@ hooks:

             (>&2 symfony-deploy)

    +crons:
    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$SYMFONY_BRANCH" = "master" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messages:
             commands:

``crons`` 的配置段定义了所有 cron 任务。每一个 cron 任务根据 ``spec`` 里的安排来执行。

``croncape`` 工具会监控命令的执行，如果一个命令返回了非 ``0`` 的退出码，该工具就会发送一封邮件到 ``MAILTO`` 环境变量定义的地址。

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

设置 ``MAILTO`` 环境变量：

.. code-block:: bash

    $ symfony var:set MAILTO=ops@example.com

在你的本地机器上，你能让一个 cron 任务强制执行：

.. code-block:: bash
    :class: ignore

    $ symfony cron comment_cleanup

请注意 cron 任务在 SymfonyCloud 的所有分支上都设置了。如果在非生产环境中你不想执行某些任务，你可以检查 ``$SYMFONY_BRANCH`` 环境变量：

.. code-block:: bash
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: 深入学习

    * `Cron/crontab 语法 <https://en.wikipedia.org/wiki/Cron>`_；

    * `Croncape 的代码仓库 <https://github.com/symfonycorp/croncape>`_；

    * `Symfony 控制台命令的文档 <https://symfony.com/doc/current/console.html>`_；

    * `Symfony 控制台速查表 <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_。
