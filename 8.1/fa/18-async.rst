پیش به سوی ناهمزمانی (Async)
=============================================

.. index::
    single: Async

بررسی هرز بودن محتوا در هنگام ارسال فرم ممکن است موجب مشکلاتی شود. اگر API  مربوط به Akismet کند شود، وب‌سایت ما نیز برای کاربران کند می‌شود. اما بدتر از آن، اگر مهلت (timeout) پایان یابد یا  API  مربوط به Akismet در دسترس نباشد، ممکن است ما کامنت‌ها را از دست بدهیم.

به صورت ایده‌آل ما باید داده‌های ارسالی را بدون انتشار آن ذخیره کنیم و به سرعت پاسخ را برگردانیم. سپس بررسی هرز بودن محتوا می‌تواند به صورت مجزا (out of band) صورت پذیرد.

پرچم‌گذاری کامنت‌ها
---------------------------------------

.. index::
    single: Command;make:entity

ما نیاز داریم که یک ``state`` برای کامنت‌ها معرفی کنیم: ``submitted``، ``spam`` و ``published``.

ویژگی ``state`` را به کلاس ``Comment`` اضافه کنید:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

migration پایگاه‌داده را ایجاد کنید:

.. code-block:: bash

    $ symfony console make:migration

migration را اصلاح کنید تا تمام کامنت‌های موجود را با مقدار پیشفرض ``published`` به‌روزرسانی نماید:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714155905 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255)');
    +        $this->addSql("UPDATE comment SET state='published'");
    +        $this->addSql('ALTER TABLE comment ALTER COLUMN state SET NOT NULL');
         }

         public function down(Schema $schema) : void

.. index::
    single: Command;doctrine:migrations:migrate

پایگاه‌داده را Migrate کنید:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

همچنین باید اطمینان یابیم که به صورت پیشفرض،‌ ``state`` با مقدار ``submitted`` تنظیم شده است:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -55,9 +55,9 @@ class Comment
         private $photoFilename;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, options={"default": "submitted"})
          */
    -    private $state;
    +    private $state = 'submitted';

         public function __toString(): string
         {

پیکربندی EasyAdmin را به‌روزرسانی کنید تا قادر به مشاهده‌ی وضعیت کامنت‌ها باشیم:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/easy_admin.yaml
    +++ b/config/packages/easy_admin.yaml
    @@ -18,6 +18,7 @@ easy_admin:
                         - author
                         - { property: 'email', type: 'email' }
                         - { property: 'photoFilename', type: 'image', 'base_path': "/uploads/photos", label: 'Photo' }
    +                    - state
                         - { property: 'createdAt', type: 'datetime' }
                     sort: ['createdAt', 'ASC']
                     filters: ['conference']
    @@ -26,5 +27,6 @@ easy_admin:
                         - { property: 'conference' }
                         - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                         - 'author'
    +                    - { property: 'state' }
                         - { property: 'email', type: 'email' }
                         - text

به‌روزرسانی آزمون‌ها با تنظیم ``state`` برای fixture‌ها را فراموش نکنید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
    @@ -37,8 +37,16 @@ class AppFixtures extends Fixture
             $comment1->setAuthor('Fabien');
             $comment1->setEmail('fabien@example.com');
             $comment1->setText('This was a great conference.');
    +        $comment1->setState('published');
             $manager->persist($comment1);

    +        $comment2 = new Comment();
    +        $comment2->setConference($amsterdam);
    +        $comment2->setAuthor('Lucas');
    +        $comment2->setEmail('lucas@example.com');
    +        $comment2->setText('I think this one is going to be moderated.');
    +        $manager->persist($comment2);
    +
             $admin = new Admin();
             $admin->setRoles(['ROLE_ADMIN']);
             $admin->setUsername('admin');

.. index::
    single: Test;Container
    single: Container;Test

برای آزمون‌های کنترلر، اعتبارسنجی را شبیه‌سازی کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -2,6 +2,8 @@

     namespace App\Tests\Controller;

    +use App\Repository\CommentRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

     class ConferenceControllerTest extends WebTestCase
    @@ -22,10 +24,16 @@ class ConferenceControllerTest extends WebTestCase
             $client->submitForm('Submit', [
                 'comment_form[author]' => 'Fabien',
                 'comment_form[text]' => 'Some feedback from an automated functional test',
    -            'comment_form[email]' => 'me@automat.ed',
    +            'comment_form[email]' => $email = 'me@automat.ed',
                 'comment_form[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
             ]);
             $this->assertResponseRedirects();
    +
    +        // simulate comment validation
    +        $comment = self::$container->get(CommentRepository::class)->findOneByEmail($email);
    +        $comment->setState('published');
    +        self::$container->get(EntityManagerInterface::class)->flush();
    +
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }

از درون یک آزمون PHPUnit، می‌توانید هر سرویسی را از کانتینر و از طریق ``self::$container->get()`` دریافت کنید؛ همچنین این روش دسترسی به سرویس‌های غیر عمومی را نیز ممکن می‌کند.

درک کردن پیغام‌رسان (Messenger)
-------------------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

مدیریت کدهای غیرهمزمان در سیمفونی بر عهده‌ی کامپوننت پیغام‌رسان (Messenger) است:

.. code-block:: bash

    $ symfony composer req messenger

زمانی که لازم باشد یک منطق به صورت غیرهمزمان اجرا گردد، یک *پیغام (message)* برای *گذرگاه پیغام‌رسان (messenger bus)* ارسال کنید. گذرگاه، پیغام را در یک *صف(queue)* ذخیره کرده و به صورت آنی باز می‌گرداند تا جریان عملیات بتوان به سریع‌ترین وجه ممکن ادامه یابد.

یک *مصرف‌کننده (consumer)* به صورت مستمر در پس‌زمینه پیغام‌های جدید درون صف را می‌خواند و منطق مربوطه را اجرا می‌کند. مصرف‌کننده می‌تواند بر روی همان سرور اپلیکیشن وب یا بر روی یک سرور مجزا اجرا گردد.

این بسیار شبیه به شیوه‌ی رسیدگی به درخواست‌های HTTP است، با این تفاوت که دیگر واکنشی نداریم.

کدنویسی یک رسیدگی‌کننده به پیغام (Message Handler)
-------------------------------------------------------------------------------

پیغام یک شیء داده‌ای است که نباید حاوی هیچ منطقی باشد. پیغام برای ذخیره‌شدن در صف، serialize می‌شود، بنابراین تنها داده‌های «ساده‌ی» قابل serialize‌کردن را ذخیره کنید.

یک کلاس ``CommentMessage`` ایجاد کنید:

.. code-block:: php
    :caption: src/Message/CommentMessage.php

    namespace App\Message;

    class CommentMessage
    {
        private $id;
        private $context;

        public function __construct(int $id, array $context = [])
        {
            $this->id = $id;
            $this->context = $context;
        }

        public function getId(): int
        {
            return $this->id;
        }

        public function getContext(): array
        {
            return $this->context;
        }
    }

در دنیای پیغام‌رسان، ما کنترلر نداریم، بلکه رسیدگی‌کنندگان به پیغام (message handlers) را داریم.

یک کلاس ``CommentMessageHandler`` که می‌داند چگونه به پیغام‌های ``CommentMessage`` رسیدگی کند را در فضای‌نام (namespace) جدید ``App\MessageHandler`` ایجاد کنید:

.. code-block:: php
    :caption: src/MessageHandler/CommentMessageHandler.php

    namespace App\MessageHandler;

    use App\Message\CommentMessage;
    use App\Repository\CommentRepository;
    use App\SpamChecker;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Component\Messenger\Handler\MessageHandlerInterface;

    class CommentMessageHandler implements MessageHandlerInterface
    {
        private $spamChecker;
        private $entityManager;
        private $commentRepository;

        public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository)
        {
            $this->entityManager = $entityManager;
            $this->spamChecker = $spamChecker;
            $this->commentRepository = $commentRepository;
        }

        public function __invoke(CommentMessage $message)
        {
            $comment = $this->commentRepository->find($message->getId());
            if (!$comment) {
                return;
            }

            if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
                $comment->setState('spam');
            } else {
                $comment->setState('published');
            }

            $this->entityManager->flush();
        }
    }

``MessageHandlerInterface`` یک رابط * نشانه‌گذار (marker)* است. این رابط تنها به سیمفونی کمک می‌کند تا به صورت خودکار کلاس را به عنوان یک پیغام‌رسان ثبت و پیکربندی کند. بر اساس قرارداد، منطق یک رسیدگی‌کننده در درون متدی که ``__invoke()`` نامیده می‌شود،‌ قرار می‌گیرد. راهنمای نوعِ (type hint) ``CommentMessage`` بر روی آرگمان این متد، به پیغام‌رسان می‌گوید که این کلاس، به چه کلاس‌هایی (چه نوع پیغام‌هایی) رسیدگی می‌کند.

کنترلر را به‌روزرسانی کنید تا از سیستم جدید استفاده کند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -5,14 +5,15 @@ namespace App\Controller;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use App\Form\CommentFormType;
    +use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;

    @@ -20,11 +21,13 @@ class ConferenceController extends AbstractController
     {
         private $twig;
         private $entityManager;
    +    private $bus;

    -    public function __construct(Environment $twig, EntityManagerInterface $entityManager)
    +    public function __construct(Environment $twig, EntityManagerInterface $entityManager, MessageBusInterface $bus)
         {
             $this->twig = $twig;
             $this->entityManager = $entityManager;
    +        $this->bus = $bus;
         }

         /**
    @@ -40,7 +43,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -58,6 +61,7 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -65,11 +69,8 @@ class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    -                throw new \RuntimeException('Blatant spam, go away!');
    -            }

    -            $this->entityManager->flush();
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

ما حالا به جای تکیه به بررسی‌کنند‌ه‌ی داده‌های هرز، یک پیغام را به گذرگاه اعزام می‌کنیم. سپس رسیدگی‌کننده تصمیم می‌گیرد که با آن چه کار کند.

ما به چیزی غیرمنتظره دست یافتیم. ما کنترلرمان را از بررسی‌کننده‌ی داده‌ی هرز (Spam Checker) مجزا کردیم و منطق آن را به یک کلاس جدید یعنی رسیدگی‌کننده (handler) انتقال دادیم. این یک نمونه‌ی عالی از کارکرد گذرگاه است. کد را بیازمایید، کار می‌کند. هنوز همه چیز به صورت همزمان کار می‌کند، اما احتمالاً الان کد «بهتر» است.

محدودسازی کامنت‌های نمایش‌داده‌شده
---------------------------------------------------------------------

منطق نمایش را به‌روزرسانی کنید تا از ظاهرشدن کامنت‌های منتشرنشده در جلوی صحنه جلوگیری شود:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -27,7 +27,9 @@ class CommentRepository extends ServiceEntityRepository
         {
             $query = $this->createQueryBuilder('c')
                 ->andWhere('c.conference = :conference')
    +            ->andWhere('c.state = :state')
                 ->setParameter('conference', $conference)
    +            ->setParameter('state', 'published')
                 ->orderBy('c.createdAt', 'DESC')
                 ->setMaxResults(self::PAGINATOR_PER_PAGE)
                 ->setFirstResult($offset)

پیش به سوی ناهمزمانی واقعی
------------------------------------------------

.. index::
    single: RabbitMQ

به صورت پیشفرض، رسیدگی‌کننده‌ها به صورت همزمان اجرا می‌شوند. برای ناهمزمان کردن، لازم دارید که در فایل پیکربندی ``config/packages/messenger.yaml``، به صورت صریح مشخص کنید که برای هر رسیدگی‌کننده باید از کدام صف استفاده شود:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,10 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            # async: '%env(MESSENGER_TRANSPORT_DSN)%'
    +            async: '%env(RABBITMQ_DSN)%'
                 # failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:
                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

پیکربندی به گذرگاه می‌گوید که نمونه‌های ``App\Message\CommentMessage`` را به صف ``async`` ارسال کند که توسط یک DSN تعریف شده و در متغیر محیط ``RABBITMQ_DSN`` ذخیره شده است.

Adding RabbitMQ to the Docker Stack
-----------------------------------

.. index::
    single: Docker;RabbitMQ

As you might have guessed, we are going to use RabbitMQ:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

Restarting Docker Services
--------------------------

To force Docker Compose to take the RabbitMQ container into account, stop the containers and restart them:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

.. sidebar:: دامپ‌کردن و بازیابی داده‌‌های پایگاه‌داده

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

مصرف‌کردن پیغام‌ها
-------------------------------------

اگر سعی کنید که یک کامنت جدید ارسال کنید، دیگر بررسی‌کننده‌ی محتوای هرز اجرا نمی‌شود. برای تأیید این امر، یک فراخوانی ``error_log()`` به متد ``getSpamScore()`` بیافزایید. به جای آن، یک پیغام در RabbitMQ در حال انتظار است و آمادگی دارد تا توسط یک پردازشگر مصرف شود.

.. index::
    single: Command;messenger:consume

همانطور که احتمالاً تصور کرده‌اید، سیمفونی یک فرمان مصرف‌کننده به همراه دارد. حالا آن را اجرا کنید:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

این فرمان باید بی‌درنگ پیغامی را برای کامنت ارسال‌شده اعزام گردیده است، مصرف کند:

.. code-block:: text
    :class: ignore

     [OK] Consuming messages from transports "async".

     // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

     // Quit the worker with CONTROL-C.

    11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
    11:30:20 INFO      [http_client] Request: "POST https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [http_client] Response: "200 https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
    11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]

فعالیت مصرف‌کننده‌ی پیغام، log شده است. اما شما با دادن پرچم ``-vv``، به صورت آنی در کنسول بازخورد می‌گیرد. حتی شما باید قادر به تشخیص فراخوانی API مربوط به Akismet نیز باشد.

برای متوقف کردن مصرف‌کننده، ``Ctrl+C`` را فشار دهید.

Exploring the RabbitMQ Web Management Interface
-----------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

If you want to see queues and messages flowing through RabbitMQ, open its web management interface:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Or from the web debug toolbar:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Use ``guest``/``guest`` to login to the RabbitMQ management UI:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

اجرای کارگرها (Workers) در پس‌زمینه
----------------------------------------------------------

به جای اینکه هر بار پس از ارسال کامنت، مصرف‌کننده را اجرا و به محض پایان اجرا، آن را متوقف کنیم، ما می‌خواهیم که آن را به صورت پیوسته و بدون آنکه لازم  به بازکردن تعداد زیادی پنجره‌ یا تب ترمینال باشد، اجرا کنیم.

رابط خط فرمان سیمفونی می‌تواند با پرچم شبه (``-d``) بر روی فرمان ``run``، چنین فرمان‌ها یا کارگر‌هایِ پس‌زمینه‌ای را اجرا کند.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

مجدداً مصرف‌کننده‌ی پیغام را اجرا کنید، اما آن را به پس‌زمینه بفرستید:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

گزینه‌ی ``--watch`` به سیمفونی می‌گوید که هرگاه تغییری از نوع فایل‌سیستم در پوشه‌های ``config/``، ``src/``، ``templates/`` یا ``vendor/`` بوجود آید، باید این فرمان را مجدداً راه‌اندازی نماید.

.. note::

    از پرچم ``-vv`` استفاده نکنید چرا که موجب پیغام‌های تکراری در ``server:log`` می‌شود (پیغام‌های لاگ‌شده و پیغام‌های کنسول).

اگر مصرف‌کننده به دلیلی (محدودیت حافظه، باگ و ...) از کار بیافتد، به صورت خودکار راه‌اندازی مجدد خواهد شد. و اگر مصرف‌کننده مجدداً خیلی سریع شکست بخورد، رابط خط فرمان سیمفونی، بازراه‌اندازی آن را رها خواهد کرد.

.. index::
    single: Symfony CLI;server:log

لاگ‌ها از طریق ``symfony server:log`` به همراه سایر لاگ‌هایی که از PHP، وب سرور و اپلیکیشن می‌آیند، جریان می‌یابند:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

از فرمان ``server:status`` استفاده کنید تا تمام کارگرهای پس‌زمینه‌ی به‌کاررفته برای پروژه‌ی فعلی لیست شوند:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

برای متوقف‌کردن یک کارگر، وب سرور را متوقف کنید یا PID داده‌شده توسط فرمان ``server:status`` را بکشید:

.. code-block:: bash
    :class: ignore

    $ kill 15774

تلاش مجدد برای پیغام‌های شکست‌خورده
--------------------------------------------------------------------

چه پیش می‌آید اگر هنگان مصرف پیغام، Akismet خراب باشد؟ برای افراد ارسال‌کننده‌ی پیغام مشکلی ایجاد نمی‌شود، اما پیغام از دست می‌رود و هرزبودن داده بررسی نمی‌گردد.

پیغام‌رسان دارای یک مکانیسم تلاش مجدد برای زمان‌هایی است که یک استثناء هنگام رسیدگی به پیغام رخ می‌دهد. بیایید آن را پیکربندی کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,17 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            async: '%env(RABBITMQ_DSN)%'
    -            # failed: 'doctrine://default?queue_name=failed'
    +            async:
    +                dsn: '%env(RABBITMQ_DSN)%'
    +                retry_strategy:
    +                    max_retries: 3
    +                    multiplier: 2
    +
    +            failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

    +        failure_transport: failed
    +
             routing:
                 # Route your messages to the transports
                 App\Message\CommentMessage: async

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

اگر هنگام رسیدگی به پیغام، مشکلی رخ دهد، مصرف‌کننده ۳ بار تلاش مجدد می‌کند و پس از آن از تلاش بیشتر دست می‌کشد. اما به جای دورانداختن پیغام، آن را در یک انبار دائمی‌تر ذخیره می‌کند. یعنی در صف ``failed`` که از پایگاه‌داده‌ی Doctrine استفاده می‌کند.

با کمک فرمان‌های زیر، پیغام‌های شکست‌خورده را بررسی کرده و مجدداً برای توفیق آن‌ها تلاش کنید:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Deploying RabbitMQ
------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Adding RabbitMQ to the production servers can be done by adding it to the list of services:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -5,3 +5,8 @@ db:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Reference it in the web container configuration as well and enable the ``amqp`` PHP extension:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - amqp
             - redis
             - pdo_pgsql
             - apcu
    @@ -26,6 +27,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

When the RabbitMQ service is installed on a project, you can access its web management interface by opening the tunnel first:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

اجرای کارگرها در SymfonyCloud
-------------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

برای مصرف پیغام‌ها از RabbitMQ، ما نیاز داریم تا فرمان ``messenger:consume`` را به صورت مستمر اجرا کنیم. در SymfonyCloud، این برای یک *کارگر (worker)*، قانون است:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -54,3 +54,8 @@ hooks:
             set -x -e

             (>&2 symfony-deploy)
    +
    +workers:
    +    messages:
    +        commands:
    +            start: symfony console messenger:consume async -vv --time-limit=3600 --memory-limit=128M

همچون رابط خط فرمان سیمفونی، اینجا نیز SymfonyCloud بازراه‌اندازی‌ها و لاگ‌ها را مدیریت می‌کند.

.. index::
    single: Symfony CLI;logs

برای گرفتن لا‌گ‌های یک کارگر، از این استفاده کنید:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: بیشتر بدانید

    * `آموزش تصویری پیغام‌رسان در SymfonyCasts <https://symfonycasts.com/screencast/messenger>`_؛

    * معماری `گذرگاه سرویس سازمانی (ESB) <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ و `الگوی CQRS <https://martinfowler.com/bliki/CQRS.html>`_؛

    * `مستندات پیغام‌رسان سیمفونی <https://symfony.com/doc/current/messenger.html>`_؛

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
