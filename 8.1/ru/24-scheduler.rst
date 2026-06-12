Планирование задач
========================================

.. index::
    single: Cron

Некоторые задачи обслуживания должны выполняться по расписанию. В отличие от воркеров, которые работают непрерывно, запланированные задачи запускаются периодически и на короткое время.

Очистка ненужных комментариев
--------------------------------------------------------

Комментарии, помеченные как спам или отклонённые администратором, хранятся в базе, чтобы администратор мог просмотреть их позже. Но они, вероятно, в любом случае должны быть удалены через определённое время. Думаю, что хранить такие комментарии в течение недели после создания будет достаточно.

Добавьте несколько вспомогательных методов в репозиторий комментариев для поиска, подсчёта и удаления отклонённых комментариев:

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

Все команды приложения регистрируются вместе со встроенными командами Symfony, и все они доступны через ``symfony console``. Поскольку количество доступных команд может быть большим, необходимо группировать их по пространствам имён. По соглашению команды приложения должны храниться в пространстве ``app``. Можно добавлять любое количество подпространств, разделяя их двоеточием (``:``).

Команда объявляет свои *аргументы* и *опции* с помощью атрибутов ``#[Argument]`` и ``#[Option]`` на параметрах метода ``__invoke()`` (параметр ``$dryRun`` становится опцией ``--dry-run``). Остальные параметры Symfony внедряет по их типу: ``SymfonyStyle`` для красиво отформатированного вывода в консоль и любой сервис, например репозиторий комментариев, так же, как и для аргументов контроллеров.

Очистите базу данных, выполнив команду:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Планирование команды
--------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Запускать команду вручную можно, но она должна выполняться каждую ночь. Компонент Symfony Scheduler генерирует сообщения по расписанию; затем они обрабатываются воркером, как и любые другие сообщения Messenger.

Добавьте компонент Scheduler вместе с библиотекой, которая разбирает cron-выражения:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Запланируйте команду с помощью атрибута ``#[AsCronTask]``:

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

Атрибут регистрирует команду в *расписании* по умолчанию с cron-выражением: каждую ночь в 23:50 (UTC). Проверьте:

.. code-block:: terminal

    $ symfony console debug:scheduler

Расписание доступно как обычный транспорт Messenger с тем же именем; обрабатывайте его, как любой другой транспорт:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Развёртывание расписания
------------------------

.. index::
    single: Upsun;Workers

В Upsun воркер обрабатывает только транспорт ``async``. Заставьте его обрабатывать и расписание:

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

Это всё, что нужно: ни crontab, ни дополнительного процесса; расписание живёт в PHP-коде, рядом с задачей, которую оно запускает, и развёртывается и версионируется вместе с остальной частью приложения.

А как же системные cron?
------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun также поддерживает cron-задания на уровне операционной системы, описываемые в ``.upsun/config.yaml`` рядом с веб-контейнером и воркерами; конфигурация по умолчанию уже определяет одно, которое очищает устаревшие PHP-сессии. Системные cron хорошо подходят для задач, не реализованных на PHP.

Утилита ``croncape``, используемая cron-заданием по умолчанию, следит за выполнением команды и отправляет письмо на адреса, указанные в переменной окружения ``MAILTO``, если команда возвращает код выхода, отличный от ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Обратите внимание, что cron-задания устанавливаются на всех ветках Upsun. Если вы не хотите запускать некоторые из них вне продакшена, проверяйте переменную окружения ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Двигаемся дальше

    * `Документация компонента Scheduler`_;

    * `Синтаксис cron/crontab`_;

    * `Репозиторий Croncape`_;

    * `Команды Symfony Console`_;

    * `Шпаргалка по Symfony Console`_.

.. _`Документация компонента Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`Синтаксис cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Репозиторий Croncape`: https://github.com/symfonycorp/croncape
.. _`Команды Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Шпаргалка по Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
