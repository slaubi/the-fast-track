Виконання завдань cron
======================================

.. index::
    single: Cron

Завдання cron корисні для виконання завдань з технічного обслуговування. На відміну від воркерів, вони працюють за розкладом, протягом короткого періоду часу.

Очищування коментарів
-----------------------------------------

Коментарі, позначені як спам або відхилені адміністратором, зберігаються в базі даних, оскільки адміністратор може захотіти оглянути їх пізніше. Але вони, ймовірно, мають бути видалені через деякий час. Мабуть, достатньо тримати їх протягом тижня після їх створення.

Створіть кілька корисних методів у репозиторії коментарів, щоб знайти відхилені коментарі, підрахувати їх і видалити:

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

    Для складніших запитів іноді корисно поглянути на згенеровані SQL-вирази (їх можна знайти в журналах і в профілювальнику для веб-запитів).

Використання констант класу, параметрів контейнера та змінних середовища
----------------------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 днів? Ми могли б вибрати інше число, можливо, 10 або 20. Це число може змінитися з часом. Ми вирішили зберегти його як константу в класі, але у нас є можливість зберегти його як параметр у контейнері, або навіть визначити його як змінну середовища.

Ось кілька правил, щоб вирішити, яку абстракцію використовувати:

* Якщо значення є чутливим (паролі, токени API, ...), використовуйте *секретне сховище* Symfony або Vault;

* Якщо значення є динамічним, і ви можете змінити його *без* повторного розгортання, використовуйте *змінну середовища*;

* Якщо значення може відрізнятися в різних середовищах, використовуйте *параметр контейнера*;

* Для всього іншого зберігайте значення в коді, наприклад, у *константі класу*.

Створення команди CLI
-------------------------------------

Видалення старих коментарів є ідеальним завданням для cron. Це слід робити на регулярній основі, а невелика затримка не надає жодного істотного впливу.

Створіть команду CLI з назвою ``app:comment:cleanup``, створивши файл ``src/Command/CommentCleanupCommand.php``:

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

Усі команди застосунку реєструються разом із вбудованими командами Symfony, і всі вони доступні за допомогою ``symfony console``. Оскільки кількість доступних команд може бути великою, вам слід групувати їх, використовуючи простори імен. За домовленістю, команди застосунку мають зберігатися в просторі імен ``app``. Додайте будь-яку кількість підпросторів імен, відокремивши їх двокрапкою (``:``).

Команда отримує *input* (аргументи й параметри, передані команді), а ви можете використовувати *output*, щоб писати в консоль.

Очистьте базу даних, виконавши команду:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Налаштування cron у Upsun
--------------------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Однією з приємних особливостей Upsun є те, що більша частина конфігурації зберігається в одному файлі: ``.platform.app.yaml``. Веб-контейнер, воркери й завдання cron описані разом, щоб полегшити обслуговування:

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

Секція ``crons`` визначає всі завдання cron. Кожне завдання виконується за ``spec`` графіком.

Утиліта ``croncape`` відстежує виконання команди й відправляє електронний лист на адреси, визначені в змінній середовища ``MAILTO``, якщо команда повертає будь-який код виходу, відмінний від ``0``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Налаштуйте змінну середовища ``MAILTO``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Зверніть увагу, що завдання cron встановлені у всіх гілках Upsun. Якщо ви не хочете запускати деякі з них у не-продакшн середовищах, перевірте змінну середовища ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Йдемо далі

    * `Синтаксис сron/сrontab`_;

    * `Репозиторій Croncape`_;

    * `Команди Symfony Console`_;

    * `Шпаргалка по Symfony Console`_.

.. _`Синтаксис сron/сrontab`: https://en.wikipedia.org/wiki/Cron
.. _`Репозиторій Croncape`: https://github.com/symfonycorp/croncape
.. _`Команди Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Шпаргалка по Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
