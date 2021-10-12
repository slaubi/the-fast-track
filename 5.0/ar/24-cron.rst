تشغيل المهمات الخلفية (Crons)
================================================

.. index::
    single: Cron

المهمات الخلفية مفيدة للقيام بمهام الصيانة. على عكس العمال (workers) ، يعملون وفق جدول زمني لفترة قصيرة من الزمن.

تنظيف التعليقات
-----------------------------

يتم الاحتفاظ بالتعليقات التي تم تمييزها كرسالة غير مرغوب فيها أو مرفوضة من قِبل المسؤول في قاعدة البيانات حيث قد يرغب المسؤول في فحصها لفترة قصيرة. ولكن ربما يجب إزالتها بعد بعض الوقت. ربما الاحتفاظ بها لمدة أسبوع بعد إنشائها كافياً.

قم بإنشاء بعض وظائف الأداة المساعدة في مستودع التعليقات (comment repository) للعثور على التعليقات المرفوضة وحسابها وحذفها:

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

    بالنسبة للاستعلامات الأكثر تعقيدًا ، يكون من المفيد أحيانًا إلقاء نظرة على عبارات SQL التي تم إنشاؤها (يمكن العثور عليها في السجلات "logs" وفي ملفات التعريف لطلبات الويب "profiler").

إستخدام ثوابت الفئة، معالم الحاوية، ومتغيرات بيئة العمل
-------------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 أيام؟ كان بإمكاننا اختيار رقم آخر ، ربما 10 أو 20. قد يتطور هذا الرقم بمرور الوقت. لقد قررنا تخزينه كملف ثابت في الملف ، يمكننا تخزينه كمعلمة في الحاوية ( container parameter)، أو بتعريفه على أنه متغير بيئة (environment variable).

فيما يلي بعض القواعد  لتحديد التجريد الذي يجب استخدامه:

* إذا كانت القيمة حساسة (كلمات سر، رموز API...)، إستخدم *المخزن السري* (*secret storage*) الخاص بسيمفوني او Vault

* إذا كانت القيمة ديناميكية ويجب أن تكون قادرًا على تغييرها *دون* إعادة النشر ، فاستخدم *متغير بيئة* ؛

* إذا كانت القيمة يمكن أن تكون مختلفة بين البيئات ، فاستخدم معلمة * حاوية * ؛

* بالنسبة إلى أي شيء آخر ، قم بتخزين القيمة في التعليمات البرمجية ، كما هو الحال في *فئة ثابت*.

إنشاء أمر CLI
---------------------

إزالة التعليقات القديمة هي المهمة المثالية لوظيفة cron. ينبغي أن يتم ذلك بشكل منتظم ، وليس للتأخير القليل أي تأثير كبير.

قم بإنشاء أمر خاص بشاشة الاوامر يُسمي ``app:comment:cleanup`` عن طريق انشاء ملف ``src/Command/CommentCleanupCommand.php``:

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

يتم تسجيل جميع اوامر التطبيق جنباً الي جنب مع الاوامر المُضمنة مع سيمفوني ويتم الوصول اليهم جميعا عن طريق ``symfony console``. بما ان عدد الاوامر المتاحة يمكن ان يكون كبيراً، يجب ان تضعهم في مساحة اسم (namespace). بالاتفاق، اوامر التطبيق يجب ان يتم تخزينها في مساحة اسم ``app``. أضف اي عدد من مساحات الاسم الفرعية عن طريق فصلهم بواسطة نقطة مزدوجة (``:``).

يحصل الامر علي *المُعطيات* (*input*) (الوسيطات والخيارات يتم تمريرها الي الامر) ويمكنك استخدام *المُخرجات* (*output*) لتكتب الي شاشة الاوامر.

تنظيف قاعدة البيانات عن طريق تشغيل الأمر:

.. code-block:: bash

    $ symfony console app:comment:cleanup

إعداد Cron على SymfonyCloud
-----------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

أحد الاشياء اللطيفة عن SymfonyCloud ان معظم الاعدادات يتم تخزينها في ملف واحد: ``.symfony.cloud.yaml``. حاوية الويب، العُمال، ووظائف الـ cron يتم وصفهم معاً للمساعدة علي الصيانة.

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

يحدد قسم "crons" جميع وظائف cron. يعمل كل كرون وفقًا لجدول "المواصفات".

تقوم اداة الـ ``croncape`` برصد عملية تنفيذ الامر وارسال بريد الكتروني الي العناوين المُعرفة في متغير بيئة العمل ``MAILTO`` لو قام الامر بإعادة اي رمز خروج غير ``0``.

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

قم بتكوين متغير البيئة `` MAILTO``:

.. code-block:: bash

    $ symfony var:set MAILTO=ops@example.com

يمكنك إجبار cron على التشغيل من جهازك المحلي:

.. code-block:: bash
    :class: ignore

    $ symfony cron comment_cleanup

يجب ان نشير الي انه تم إعداد crons علي كل فروع SymfonyCloud. إن كنت لا تريد ان تقوم بتشغيل بعضها علي بيئات عمل غير انتاجية، تحقق من متغير بيئة عمل ``$SYMFONY_BRANCH``.

.. code-block:: bash
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: الذهاب أبعد من ذلك

    * `Cron/crontab تركيب الجمل <https://en.wikipedia.org/wiki/Cron>`_؛

    * `Croncape repository <https://github.com/symfonycorp/croncape>`_؛

    * `Symfony Console أوامر <https://symfony.com/doc/current/console.html>`_؛

    * The `Symfony Console Cheat Sheet <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_.
