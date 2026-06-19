Scheduling Tasks
================

.. index::
    single: Cron

Some maintenance tasks must run on a schedule. Unlike workers, which run continuously, scheduled tasks run periodically for a short period of time.

Cleaning up Comments
--------------------

Comments marked as spam or rejected by the admin are kept in the database as the admin might want to inspect them for a little while. But they should probably be removed after some time. Keeping them around for a week after their creation is probably enough.

Create some utility methods in the comment repository to find rejected comments, count them, and delete them:

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

All application commands are registered alongside Symfony built-in ones and they are all accessible via ``symfony console``. As the number of available commands can be large, you should namespace them. By convention, the application commands should be stored under the ``app`` namespace. Add any number of sub-namespaces by separating them by a colon (``:``).

A command declares its *arguments* and *options* with the ``#[Argument]`` and ``#[Option]`` attributes on the parameters of ``__invoke()`` (the ``$dryRun`` parameter becomes the ``--dry-run`` option). Symfony injects the other parameters based on their type: ``SymfonyStyle`` to write nicely-formatted output to the console, and any service, like the comment repository, the same way it does for controller arguments.

Clean up the database by running the command:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Scheduling the Command
----------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Running the command by hand works, but it should run every night. The Symfony Scheduler component generates messages on a schedule; they are then consumed by a worker, like any other Messenger messages.

Add the Scheduler component, along with the library that parses cron expressions:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Schedule the command with the ``#[AsCronTask]`` attribute:

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

The attribute registers the command on the default *schedule* with a cron expression: every night at 11.50 pm (UTC). Check it:

.. code-block:: terminal

    $ symfony console debug:scheduler

A schedule is exposed as a regular Messenger transport named after it; consume it like any other transport:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Deploying the Schedule
----------------------

.. index::
    single: Upsun;Workers

On Upsun, the worker only consumes the ``async`` transport. Make it consume the schedule as well:

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

That's all it takes: no crontab, no extra process; the schedule lives in the PHP code, next to the task it triggers, and it is deployed and versioned like the rest of the application.

What about System Crons?
------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun also supports OS-level cron jobs, described in ``.upsun/config.yaml`` alongside the web container and the workers; the default configuration already defines one that cleans up expired PHP sessions. System crons are a good fit for tasks that are not implemented in PHP.

The ``croncape`` utility used by the default cron monitors the execution of the command and sends an email to the addresses defined in the ``MAILTO`` environment variable if the command returns any exit code different than ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Note that crons are set up on all Upsun branches. If you don't want to run some on non-production environments, check the ``$PLATFORM_ENVIRONMENT_TYPE`` environment variable:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Going Further

    * The `Scheduler component docs`_;

    * `Cron/crontab syntax`_;

    * `Croncape repository`_;

    * `Symfony Console commands`_;

    * The `Symfony Console Cheat Sheet`_.

.. _`Scheduler component docs`: https://symfony.com/doc/current/scheduler.html
.. _`Cron/crontab syntax`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console commands`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
