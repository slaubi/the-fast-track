اجرای وظایف زمانبندی‌شده (Cron-Jobs)
===========================================================

.. index::
    single: Cron

وظایف زمان‌بندی‌شده برای انجام وظایف نگهداری (maintenance) مفید هستند. برخلاف کارگرها، آن‌ها در یک زمان‌بندی خاص و برای مدت کوتاهی اجرا می‌شوند.

پاکسازی کامنت‌ها
--------------------------------

کامنت‌هایی که توسط مدیر به عنوان داده‌ی هرز یا ردشده علامت‌گذاری شده‌اند، در پایگاه‌داده نگه‌داشته می‌شوند چرا که ممکن است مدیر بخواهد آنها را برای مدتی مورد بررسی قرار دهد. اما احتمالاً باید پس از مدتی حذف شوند. احتمالاً نگهداری آنها تا یک هفته پس از زمان ایجادشان کافی است.

در مخزن کامنت، تعدادی متد کاربردی برای پیدا کردن کامنت‌های رد‌شده، شمارش آن‌ها و پاک‌کردن آن‌ها ایجاد کنید:

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

    برای پرس‌وجو‌های پیچیده‌تر، گاهی اوقات مفید است که به بیانیه‌ی SQL تولید‌شده نگاهی بیاندازیم (می‌توانید آن‌ را در لاگ‌ها و یا برای درخواست‌های وب در نمایه‌ساز نیز پیدا کنید).

استفاده از ثابت‌های کلاس، پارامترهای کانتینر و متغیرهای محیط
-----------------------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

۷ روز؟ می‌توانستیم عدد دیگری انتخاب کنیم، شاید ۱۰ یا ۲۰. این عدد ممکن است در طول زمان تکامل یابد. ما تصمیم گرفتیم که این عدد را به صورت یک ثابت در کلاس ذخیره کنیم، اما ممکن بود که آن را به عنوان پارامتر در کانتینر ذخیره کنیم یا حتی آن را به عنوان متغیر محیط تعریف کنیم.

اینها تعدادی قاعده‌ی قابل اتکا هستند که می‌توانید از آن برای انتخاب سطح مناسب انتزاع استفاده کنید:

* اگر مقدار حساس است (رمزعبورها، توکن‌های API و ...)، از *انبار رمز (secret storage)* سیمفونی یا یک گاوصندوق استفاده کنید؛

* اگر مقدار پویا است و باید قادر باشید که مقدار آن را بدون استقرار مجدد تغییر دهید، از *متغیر محیط* استفاده کنید؛

* اگر مقدار می‌تواند بین محیط‌های مختلف، متفاوت باشد، از یک *پارامتر کانتینر* استفاده کنید؛

* برای هر چیز دیگری، مقدار را در کد (مثلاً در یک *ثابت کلاس*) ذخیره کنید.

ایجاد یک فرمان CLI
------------------------------

حذف کامنت‌های قدیمی، یک تکلیف مناسب برای یک وظیفه‌ی زمانبندی‌شده است. این کار باید به طور منظم اجرا شود و مقداری تأخیر، تأثیر بسزایی بر روی آن ندارد.

با ایجاد فایل ``src/Command/CommentCleanupCommand.php``، یک فرمان CLI با نام ``app:comment:cleanup`` ایجاد کنید:

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

تمام فرمان‌های اپلیکیشن، در کنار فرمان‌های داخلی سیمفونی ثبت می‌شوند و از طریق ``symfony console`` در دسترس هستند. از آنجایی که تعداد فرمان‌های در دسترس می‌تواند زیاد باشد، شما باید آن‌ها را در فضای نام (namespace) دسته‌بندی کنید. بر اساس قرارداد، فرمان‌های اپلیکیشن باید در زیر فضای نام ``app`` ذخیره شوند. با جداکردن زیرفضاهای نام از طریق دونقطه (``:``)، هر تعداد که می‌خواهید زیرفضای نام اضافه کنید.

یک فرمان، *ورودی* (آرگمان‌ها و گزینه‌های داده‌شده به فرمان) می‌گیرد و شما می‌تواند از *خروجی* استفاده کنید تا در کنسول بنویسید.

با اجرای فرمان، پایگاه‌داده را پاکسازی کنید:

.. code-block:: bash

    $ symfony console app:comment:cleanup

تنظیم یک وظیفه‌ی زمان‌بندی‌شده در SymfonyCloud
------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

یکی از خوبی‌های SymfonyCloud این است که اکثر پیکربندی در یک فایل ذخیره می‌شود: ``.symfony.cloud.yaml``. کانتینر وب، کارگرها و وظایف زمان‌بندی‌شده کنارهم توصیف شده‌اند تا به نگهداری آن‌ها کمک شود:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -56,6 +56,15 @@ hooks:

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

بخش ``crons`` تمام وظایف زمان‌بندی‌شده را تعریف می‌کند. هر وظیفه‌ی زمان‌بندی‌شده، با توجه به زمان‌بندی  ``spec`` اجرا می‌شود.

ابزار کاربردی ``croncape``، بر اجرای فرمان نظارت می‌کند و اگر فرمان هر کد خروجی به جز ``0`` برگرداند، به آدرس‌هایی که در متغیر محیط ``MAILTO`` تعریف شده است، یک رایانامه ارسال می‌کند.

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

متغیر محیط ``MAILTO`` را پیکربندی کنید:

.. code-block:: bash

    $ symfony var:set MAILTO=ops@example.com

شما می‌توانید یک وظیفه‌ی زمان‌بندی‌شده را مجبور کنید که از ماشین محلی‌تان اجرا شود:

.. code-block:: bash
    :class: ignore

    $ symfony cron comment_cleanup

توجه کنید که وظایف زمان‌بندی‌شده بر روی تمام شاخه‌های SymfonyCloud تنظیم شده‌اند. اگر نمی‌خواهید که برخی از آن‌ها را در محیط‌های غیر عمل‌آوری اجرا کنید، متغیر محیط ``$SYMFONY_BRANCH`` را بررسی کنید:

.. code-block:: bash
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: بیشتر بدانید

    * `نحو Cron/crontab <https://en.wikipedia.org/wiki/Cron>`_؛

    * `مخزن Croncape <https://github.com/symfonycorp/croncape>`_؛

    * `فرمان‌های کنسول سیمفونی <https://symfony.com/doc/current/console.html>`_؛

    * `برگه‌تقلب کنسول سیمفونی <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_.
