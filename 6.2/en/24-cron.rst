Running Crons
=============

.. index::
    single: Cron

Crons are useful to do maintenance tasks. Unlike workers, they run on a schedule for a short period of time.

Cleaning up Comments
--------------------

Comments marked as spam or rejected by the admin are kept in the database as the admin might want to inspect them for a little while. But they should probably be removed after some time. Keeping them around for a week after their creation is probably enough.

Create some utility methods in the comment repository to find rejected comments, count them, and delete them:

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
    @@ -18,6 +19,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -25,6 +28,29 @@ class CommentRepository extends ServiceEntityRepository
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
    +                'date' => new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    For more complex queries, it is sometimes useful to have a look at the generated SQL statements (they can be found in the logs and in the profiler for Web requests).

Using Class Constants, Container Parameters, and Environment Variables
----------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 days? We could have chosen another number, maybe 10 or 20. This number might evolve over time. We have decided to store it as a constant on the class, but we might have stored it as a parameter in the container, or we might have even defined it as an environment variable.

Here are some rules of thumb to decide which abstraction to use:

* If the value is sensitive (passwords, API tokens, ...), use the Symfony *secret storage* or a Vault;

* If the value is dynamic and you should be able to change it *without* re-deploying, use an *environment variable*;

* If the value can be different between environments, use a *container parameter*;

* For everything else, store the value in code, like in a *class constant*.

Creating a CLI Command
----------------------

Removing the old comments is the perfect task for a cron job. It should be done on a regular basis, and a little delay does not have any major impact.

Create a CLI command named ``app:comment:cleanup`` by creating a ``src/Command/CommentCleanupCommand.php`` file:

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

All application commands are registered alongside Symfony built-in ones and they are all accessible via ``symfony console``. As the number of available commands can be large, you should namespace them. By convention, the application commands should be stored under the ``app`` namespace. Add any number of sub-namespaces by separating them by a colon (``:``).

A command gets the *input* (arguments and options passed to the command) and you can use the *output* to write to the console.

Clean up the database by running the command:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Setting up a Cron on Platform.sh
--------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

One of the nice things about Platform.sh is that most of the configuration is stored in one file: ``.platform.app.yaml``. The web container, the workers, and the cron jobs are described together to help maintenance:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -61,6 +61,14 @@ crons:
             spec: '50 23 * * *'
             cmd: if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then croncape php-security-checker; fi

    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messenger:
             commands:

The ``crons`` section defines all cron jobs. Each cron runs according to a ``spec`` schedule.

The ``croncape`` utility monitors the execution of the command and sends an email to the addresses defined in the ``MAILTO`` environment variable if the command returns any exit code different than ``0``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Configure the ``MAILTO`` environment variable:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Note that crons are set up on all Platform.sh branches. If you don't want to run some on non-production environments, check the ``$PLATFORM_ENVIRONMENT_TYPE`` environment variable:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Going Further

    * `Cron/crontab syntax`_;

    * `Croncape repository`_;

    * `Symfony Console commands`_;

    * The `Symfony Console Cheat Sheet`_.

.. _`Cron/crontab syntax`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console commands`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
