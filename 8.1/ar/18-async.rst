الذهاب المتزامن
=============================

.. index::
    single: Async

التحقق من البيانات التطفلية (Spam) أثناء معالجة الاستمارة (الفورم Form) قد يؤدي لبعض المشاكل. لو ان ال  Akismet API  اصبحت بطيئة، كذلك موقعنا سوف يصبح بطئ بالنسبة للمستخدمين. لكن أسوء من ذلك، اذا وصلنا إلي وقت مستقطع (timeout) أو ال Akismet API أصبحت غير متاحة، من الممكن ان نخسر التعليقات (comments).

بصورة مثالية، يجب علينا حفط البيانات المقدمة دون نشرها، وإرسال رد في الحال. والتحقق من البيانات التطفلية (spam) يمكن أن يتم خارج النطاق (out of band).

إبراز التعليقات
-----------------------------

.. index::
    single: Command;make:entity

نحتاج إلي تقديم حالة ``state`` للتعليقات مثل: `submitted``, ``spam``, و ``published``.

أضف خاصية (property) الحالة `state`` إلي فئة (كلاس) التعليقات ``Comment``.

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

قم بإنشاء database migration:

.. code-block:: terminal

    $ symfony console make:migration

قم بتعديل ال migration لتحديث جميع التعليقات الموجودة لتصبح ``published`` بشكل تلقائي (default):

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255)');
    +        $this->addSql("UPDATE comment SET state='published'");
    +        $this->addSql('ALTER TABLE comment ALTER COLUMN state SET NOT NULL');
         }

         public function down(Schema $schema): void

.. index::
    single: Command;doctrine:migrations:migrate

قم بتحديث قاعدة البيانات (Migrate the database):

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

وينبغي علينا أن نتأكد من أن الحالة  ``state`` أصبحت ``submitted`` بشكل أساسي (افتراضي default):

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

قم بتعديل اعدادات ال EasyAdmin لكي تتمكن من رؤية حالة التعليقات:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
    @@ -51,6 +51,7 @@ class CommentCrudController extends AbstractCrudController
                 ->setLabel('Photo')
                 ->onlyOnIndex()
             ;
    +        yield TextField::new('state');

             $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
                 'html5' => true,

لا تنسَ أيضاً تعديل الإختبارات (tests) عن طريق ضبط الحالة ``state`` في التركيبات (المثبتات fixtures):

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

بالنسبة لاختبارات الموجهات (controller)، قم بمحاكاة التحقق من صحة البيانات (simulate the validation):

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

من اختبارات ال PHPUnit، يمكنك الحصول علي اي خدمة (service) من ال container عن طريق ``self::$container->get()``; كما أنه يمكنك الحصول علي الخدمات الغير عامة (non-public services).

فهم ال Messenger
---------------------

.. index::
    single: Messenger
    single: Components;Messenger

إدارة التعليمات البرمجية (code) الغير متزامنة (asynchronous) في سيمفوني Symfony تعتبر وظيفة مكون ال Messenger:

.. code-block:: terminal

    $ symfony composer req messenger

عندما ينبغي تنفيذ بعض المنطق (logic) الغير متزامن (asynchronously)، أرسل *رسالة* إلي ناقل الرسول *messenger bus*. يقوم هذا الناقل بتخزين الرسالة في *صف* (قائمة انتظار *queue*) ويعود في الحال ليسمح باستئناف باقي العمليات بأسرع ما يمكن.

يعمل *المستهلك* (*consumer*) بشكل مستمر في الخلفية ليقوم بقرائة الرسائل من قائمة الانتظار (queue) ويقوم بتنفيذ المنطق المرتبط بهذه الرسائل (associated logic). ويمكن للمستهلك أن يعمل علي نفس الخادم (server) الذي يعمل عليه تطبيق الويب أو علي خادم منفصل.

ويشبه جداً الطريقة التي يتم بها معالجة طلبات ال HTTP، بإستثناء أنه ليس لدينا ردود فعل (responses).

برمجة معالج (Handler) الرسالة
----------------------------------------------

لا يجب ان تحتوي الرسالة علي اي منطق (logic) فهي عبارة عن فئة كائن بيانات (data object class). سوف يتم تفكيكها (serialized) ليتم تخزينها في قائمة انتظار، لذلك قم بتخزين بيانات "بسيطة" قابلة للتفكيك (serializable).

قم بإنشاء فئة ال ``CommentMessage``:

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

في عالم الرسول (Messenger)، لا نملك موجهات (متحكمات controllers)، ولكن معالجات (handlers) رسائل.

قم بإنشاء ال `CommentMessageHandler`` تحت مساحة إسم (namespace) جديدة تسمي ``App\MessageHandler`` تعرف كيفية التعامل مع رسائل ال ``CommentMessage``.

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

يعتبر ال ``MessageHandlerInterface`` علامة (*marker*) واجهة (interface). فإنه فقط يساعد سيمفوني Symfony علي عمل تسجيل تلقائي (auto-register) والتكوين (الاعداد) التلقائي (auto-configure) للفئة (class) كمعالج للمرسال (Messenger handler). حسب الإتفاق، فإن منطق (logic) المعالج (handle) يعيش في منهج (method) يسمي ``__invoke()``. تلميح (type hint) ال ``CommentMessage`` الموجود علي واحد من ال argument لهذا المنهج (method) يخبر المرسال (Messenger) أي فئة (class) سيقوم بعالجتها.

قم بتعديل الموجهات (المتحكمات controller) لاستخدام النظام الجديد:

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

         #[Route('/', name: 'homepage')]
    @@ -36,7 +39,7 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -54,6 +57,7 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -61,11 +65,8 @@ class ConferenceController extends AbstractController
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

بدلاً من الإعتماد علي Spam Checker، فإننا الأن نرسل رسالة إلي الناقل (bus). ومن ثم يقرر المعالج ماذا يفعل معها.

لقد حققنا شئ غير متوقع. لقد قمنا بنقل المنطق (logic) لفئة جديدة وفصلنا الموجه (المتحكم controller) عن ال Spam Checker، وهو المعالج (handler). إنها حالة استخدام مثالية للناقل. إختبر الكود، إنه يعمل. كل شئ مازال يتم بشكل متزامن، ولكن الكود ربما يكون *أفضل* بالفعل.

تقييد (التحكم) في التعليقات المعروضة
------------------------------------------------------------------

قم بتعديل الكود المستخدم في إظهار التعليقات لتجنب ظهور التعليقات الغير منشورة في الواجهة الامامية (frontend):

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

الذهاب المتزامن في الحقيقة
-------------------------------------------------

بشكل تلقائي، يتم النداء علي المعالجات بالتزامن. لاستخدامها بشكل غير متزامن، يجب عليك تهيئة (إعداد) أي صف (قائمة انتظار) لكل معالج في  ملف الاعدادات الموجود في ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.env
    +++ b/.env
    @@ -29,7 +29,7 @@ DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"

     ###> symfony/messenger ###
     # Choose one of the transports below
    -# MESSENGER_TRANSPORT_DSN=doctrine://default
    +MESSENGER_TRANSPORT_DSN=doctrine://default
     # MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages
     # MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages
     ###< symfony/messenger ###
    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,15 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            # async: '%env(MESSENGER_TRANSPORT_DSN)%'
    +            async:
    +                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                options:
    +                    auto_setup: false
    +                    use_notify: true
    +                    check_delayed_interval: 60000
                 # failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:
                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

ملف الاعدادت يخبر الناقل أن يرسل حالات (أمثلة instances) من ``App\Message\CommentMessage`` الي الصف الخاص بها الذي تم تعرفيه عن طريق DSN، المخزن في متغير بيئة العمل المسمي ``RABBITMQ_DSN``.

إعداد جداول ومشغلات PostgreSQL:

.. code-block:: terminal

    $ symfony console make:migration

وترحيل قاعدة البيانات:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. tip::

    وراء الكواليس، تستخدم سيمفوني نظام النشر/النشر الفرعي المدمج والأداء PostgreSQL والقابل للتطوير والمعاملات الخاصة ب( ``LISTEN`` / ``NOTIFY``). يمكنك أيضًا قراءة فصل RabbitMQ إذا كنت تريد استخدامه بدلاً من PostgreSQL كوسيط للرسائل.

إستهلاك الرسائل
-----------------------------

إذا حاولت إرسال تعليق جديد، لن يتم إستدعاء (إستخدام) محقق البيانات الفضولية (العشوائية spam checker) بعد الان. قم بإضافة ``error_log()`` في منهج (method) ال ``getSpamScore()`` للتأكيد. بدلاً من ذلك، هناك رسالة منتظرة في الانتظار queue، مستعدة ليتم استهلاكها (مناداتها) عن طريق بعض العمليات.

.. index::
    single: Command;messenger:consume

كما قد تخيلت، فإن سيمفوني يأتي معه أمر خاص بالمستهلك. قم بإستخدامه الأن:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

سيقوم علي الفور بإستهلاك الرسالة المرسلة الخاصة بالتعليق الذي تم إرساله:

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

عملية إستهلاك الرسالة تم تسجيلها، ولكن تحصل علي رد فوري علي وحدة التحكم (الكونسل) عن طريق تمرير أمر ال ``-vv``. يمكنك حتي تحديد المناداة ل Akismet API.

لإقاف المستهلك، قم بضغط ``Ctrl+C``.

تشغيل العمال (Workers) في الخلفية
-----------------------------------------------------

بدلا من إطلاق المستهلك كل مرة نقوم بإضافة تعليق وإقافه فورا بعد ذلك، نريد ان نجعله يعمل بشكل مستمر بدون الحاجة الي العديد من شاشات التيرمنل او تبويبات (tabs) مفتوحة.

السيمفوني CLI يمكن ادارة مثل هذه الاوامر في الخلفية عن طريق اضافة علامة الشيطان (daemon flag) الي الامر عند التشغيل ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

قم بتشغيل مستهلك الرسالة مرة اخري، ولكن ارسله الي الخلفية:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

خيار ``--watch`` يخبر سيمفوني بانه عليه ان يقوم بإعادة تشغيل الامر كلما كان هناك تغير في اي ملف يقع تحت ال ``config/``، ``src/``، ``templates/``، او ``vendor/``.

.. note::

    لا تقم باستخدام ``-vv`` لانه سوف تجد رسائل متكررة في ``server:log`` (الرسائل المسجلة ورسائل وحدة التحكم (الكونسل)).

اذا توقف المستهلك عن العمل لاي سبب (حدود الذاكرة، مشكلة، ...)، سيتم إعادة تشغيله تلقائياً. ولو فشل المستهلك بسرعة أكبر مما يجب، سيمفوني CLI سوف يستسلم.

.. index::
    single: Symfony CLI;server:log

سوف تتدفق السجلات عن طريق ``symfony server:log`` مع جميع السجلات القادمة من PHP، خادم الويب، والتطبيق (الابلاكيشن):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

استخدم أمر ال ``server:status`` لسرد جميع العمال بالخلفية الذي يتم ادارتهم عن طريق المشروع الحالي:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

لإقاف احد العمال، قم باقاف خادم الويب او اقتل ال PID المعطي عن طريق امر ال ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

إعادة محاولة الرسائل الفاشلة
-----------------------------------------------------

ماذا  لو ان ال Akismet سقط (وقع) أثناء استهلاك رسالة؟ لا يوجد اي تأثير علي الاشخاص الذين يقدمون (يرسلون) التعليقات، لكن الرسالة ضاعت ولم يتم التحقق من البيانات الفضولية.

الرسول (ماسنجر Messenger) يحتوي علي آلية إعادة المحاولة عندم تحدث مشكلة أثناء معالجة الرسالة. هيا نقوم بإعداده:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -1,7 +1,7 @@
     framework:
         messenger:
             # Uncomment this (and the failed transport below) to send failed messages to this transport for later handling.
    -        # failure_transport: failed
    +        failure_transport: failed

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    @@ -11,7 +11,10 @@ framework:
                         auto_setup: false
                         use_notify: true
                         check_delayed_interval: 60000
    -            # failed: 'doctrine://default?queue_name=failed'
    +                retry_strategy:
    +                    max_retries: 3
    +                    multiplier: 2
    +            failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

لو أن مشكلة حدثت أثناء معالجة الرسالة، المستهلك سيقوم بإعادة المحاولة ثلاث مرات قبل أن يستسلم. ولكن بدلاً من تجاهل الرسالة، سوف يقوم بتخزينها بشكل دائم في قائمة انتظار ال ``failed``، التي تستخدم جدول مختلف من قاعدة البيانات.

إفحص الرسائل الفاشلة وقم باعادة محاولة إرسالهم عن طريق الاوامر التالية:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

تشغيل العمال علي SymfonyCloud
-------------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

لإستهلاك الرسائل من PostgreSQL، نحتاج الي تشغيل امر ال ``messenger:consume`` بشكل مستمر. علي SymfonyCloud، هذا هو دور *العامل*:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -50,3 +50,8 @@ hooks:
             set -x -e

             (>&2 symfony-deploy)
    +
    +workers:
    +    messages:
    +        commands:
    +            start: symfony console messenger:consume async -vv --time-limit=3600 --memory-limit=128M

مثل سيمفوني CLI، يقوم SymfonyCloud يقوم بادارة عمليات إعادة التشغيل والسجلات.

.. index::
    single: Symfony CLI;logs

للحصول علي سجلات العامل، قم بأستخدام:

.. code-block:: terminal
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: الذهاب أبعد من ذلك

    * الدروس التعليمية الخاصة بالرسول:  `SymfonyCasts Messenger tutorial <https://symfonycasts.com/screencast/messenger>`_؛

    * تصميم ناقل الخدمات المؤسسة ووتيرة CQRS: `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_  and `CQRS  <https://martinfowler.com/bliki/CQRS.html>`_؛

    * مراجع رسول سيمفوني: `Symfony Messenger docs <https://symfony.com/doc/current/messenger.html>`_؛
