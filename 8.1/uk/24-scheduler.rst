Планування завдань
==================

.. index::
    single: Cron

Деякі завдання з технічного обслуговування мають виконуватися за розкладом. На відміну від воркерів, які працюють безперервно, заплановані завдання виконуються періодично, протягом короткого періоду часу.

Очищування коментарів
-----------------------------------------

Коментарі, позначені як спам або відхилені адміністратором, зберігаються в базі даних, оскільки адміністратор може захотіти оглянути їх пізніше. Але вони, ймовірно, мають бути видалені через деякий час. Мабуть, достатньо тримати їх протягом тижня після їх створення.

Створіть кілька корисних методів у репозиторії коментарів, щоб знайти відхилені коментарі, підрахувати їх і видалити:

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

Усі команди застосунку реєструються разом із вбудованими командами Symfony, і всі вони доступні за допомогою ``symfony console``. Оскільки кількість доступних команд може бути великою, вам слід групувати їх, використовуючи простори імен. За домовленістю, команди застосунку мають зберігатися в просторі імен ``app``. Додайте будь-яку кількість підпросторів імен, відокремивши їх двокрапкою (``:``).

Команда оголошує свої *аргументи* та *параметри* за допомогою атрибутів ``#[Argument]`` і ``#[Option]`` на параметрах методу ``__invoke()`` (параметр ``$dryRun`` стає параметром ``--dry-run``). Symfony впроваджує інші параметри на основі їхнього типу: ``SymfonyStyle`` для запису гарно відформатованого виводу в консоль, а також будь-який сервіс, як-от репозиторій коментарів, так само, як і для аргументів контролера.

Очистьте базу даних, виконавши команду:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Планування команди
------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Запуск команди вручну працює, але вона має виконуватися щоночі. Компонент Symfony Scheduler генерує повідомлення за розкладом; потім вони споживаються воркером, як і будь-які інші повідомлення Messenger.

Додайте компонент Scheduler разом із бібліотекою, що парсить вирази cron:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Заплануйте команду за допомогою атрибута ``#[AsCronTask]``:

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

Атрибут реєструє команду в розкладі за замовчуванням із виразом cron: щоночі о 23:50 (UTC). Перевірте це:

.. code-block:: terminal

    $ symfony console debug:scheduler

Розклад надається як звичайний транспорт Messenger з тією ж назвою; споживайте його, як і будь-який інший транспорт:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Розгортання розкладу
--------------------

.. index::
    single: Upsun;Workers

На Upsun воркер споживає лише транспорт ``async``. Зробіть так, щоб він споживав і розклад:

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

Це все, що потрібно: жодного crontab, жодного додаткового процесу; розклад живе в PHP-коді, поруч із завданням, яке він запускає, і розгортається та версіонується, як і решта застосунку.

Що щодо системних cron?
-----------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun також підтримує завдання cron на рівні операційної системи, описані у ``.upsun/config.yaml`` поруч із веб-контейнером і воркерами; конфігурація за замовчуванням уже визначає одне, яке очищає застарілі сесії PHP. Системні cron добре підходять для завдань, які не реалізовані в PHP.

Утиліта ``croncape``, що використовується cron за замовчуванням, відстежує виконання команди й відправляє електронний лист на адреси, визначені в змінній середовища ``MAILTO``, якщо команда повертає будь-який код виходу, відмінний від ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Зверніть увагу, що завдання cron встановлені у всіх гілках Upsun. Якщо ви не хочете запускати деякі з них у не-продакшн середовищах, перевірте змінну середовища ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Йдемо далі

    * `Документація компонента Scheduler`_;

    * `Синтаксис сron/сrontab`_;

    * `Репозиторій Croncape`_;

    * `Команди Symfony Console`_;

    * `Шпаргалка по Symfony Console`_.

.. _`Документація компонента Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`Синтаксис сron/сrontab`: https://en.wikipedia.org/wiki/Cron
.. _`Репозиторій Croncape`: https://github.com/symfonycorp/croncape
.. _`Команди Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Шпаргалка по Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
