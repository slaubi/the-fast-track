Выполнение заданий cron
========================================

.. index::
    single: Cron

Задания cron полезны для задач администрирования. В отличие от воркеров, они запускаются на короткое время по расписанию.

Очистка ненужных комментариев
--------------------------------------------------------

Комментарии, помеченные как спам или отклонённые администратором, хранятся в базе, чтобы администратор мог просмотреть их позже. Но они, вероятно, в любом случае должны быть удалены через определённое время. Думаю, что хранить такие комментарии в течение недели после создания будет достаточно.

Добавьте несколько вспомогательных методов в репозиторий комментариев для поиска, подсчёта и удаления отклонённых комментариев:

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

    Для более сложных запросов иногда полезно посмотреть сгенерированные SQL-запросы (которые можно найти в логах и в профилировщике веб-запросов).

Использование констант класса, параметров контейнера и переменных среды окружения
---------------------------------------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

Почему именно 7 дней? Мы могли бы выбрать другое число, может быть 10 или 20. Это число потом может поменяться. Поэтому лучше всего хранить такие данные в константе класса, что мы и сделали, хотя это значение можно было поместить в параметр контейнера или даже определить соответствующую переменную окружения.

Несколько основных правил, по которым можно определить, какую абстракцию использовать:

* Если значение является конфиденциальной информацией (пароли, токены API и т.д.), используйте *секретное хранилище* Symfony или Vault;

* Если значение динамическое, которое должно изменяться *без* повторного развёртывания, используйте *переменные окружения*;

* Если значение может различаться в разных окружениях, используйте *параметры контейнера*;

* Во всех остальных случаях храните значение в коде, например, в *константах класса*.

Создание CLI-команды
-----------------------------------

Удаление старых комментариев — идеальное задание для cron. Оно должно выполняться регулярно и небольшие задержки в выполнении не играют существенной роли.

Создайте CLI-команду с названием ``app:comment:cleanup`` в файле ``src/Command/CommentCleanupCommand.php``:

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

Все команды приложения регистрируются вместе со встроенными командами Symfony, и все они доступны через ``symfony console``. Поскольку количество доступных команд может быть большим, необходимо группировать их по пространствам имён. По соглашению команды приложения должны храниться в пространстве ``app``. Можно добавлять любое количество подпространств, разделяя их двоеточием (``:``).

Команда принимает *input* (аргументы и параметры, переданные команде), а также *output*, который вы можете использовать для вывода данных в консоль.

Очистите базу данных, выполнив команду:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Настройка cron в Platform.sh
--------------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

Одним из удобств Platform.sh является то, что большая часть конфигурации хранится в одном файле: ``.platform.app.yaml``. Веб-контейнер, воркеры и cron-задания для простоты администрирования описаны в одном месте:

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

Раздел ``crons`` описывает все задания cron. Каждое задание выполняется в соответствии с расписанием, указанным в свойстве ``spec``.

Утилита ``croncape`` следит за выполнением задания и посылает электронное письмо на адреса, указанные в переменной окружения ``MAILTO``, если команда возвращает код завершения, отличный от ``0``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Настройте переменную окружения ``MAILTO``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Обратите внимание, что задания для cron настроены для всех веток Platform.sh. Если вы не хотите запускать какие-то задания вне продакшена, проверьте переменную окружения ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Двигаемся дальше

    * `Синтаксис cron/crontab`_;

    * `Репозиторий Croncape`_;

    * `Команды Symfony Console`_;

    * `Шпаргалка по Symfony Console`_.

.. _`Синтаксис cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Репозиторий Croncape`: https://github.com/symfonycorp/croncape
.. _`Команды Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Шпаргалка по Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
