اطلاع‌رسانی با تمام قوا
============================================

اپلیکیشن Guestbook، بازخوردهای مربوط به کنفرانس‌ها را جمع‌آوری می‌کند. اما ما در بازخورددادن به کاربرانمان عالی نیستیم.

As comments are moderated, they probably don't understand why their comments are not published instantly. They might even re-submit them thinking there was some technical problems. Giving them feedback after posting a comment would be
great.

علاوه بر این، احتمالاً باید وقتی کامنت کاربران منتشر شد به آن‌ها اطلاع دهیم. ما از آن‌ها رایانامه درخواست می‌کنیم تا بتوانیم از آن استفاده کنیم.

راه‌های زیادی برای اطلاع‌رسانی به کاربران وجود دارد. رایانامه اولین گزینه‌ای است که ممکن است به آن فکر کنید، اما اعلان‌ها (notifications) در اپلیکیشن وب نیز یک گزینه‌ی دیگر است. حتی ما می‌توانیم به ارسال پیامک (SMS) و یا ارسال یک پیغام به Slack یا Telegram فکر کنیم. گزینه‌های زیادی وجود دارد.

.. index::
    single: Components;Notifier
    single: Notifier

کامپوننت اعلانگر (Notifier) سیمفونی تعداد زیادی راهبرد اطلاع‌رسانی را پیاده‌سازی می‌کند:

.. code-block:: bash

    $ symfony composer req notifier

ارسال اعلان‌های اپلیکیشن وب در مرورگر
----------------------------------------------------------------------

.. index::
    single: Flash Messages

به عنوان اولین گام، بیایید در مرورگر پس از ارسال کامنت، به کاربران اعلام کنیم که کامنت‌ها تعدیل می‌شوند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -14,6 +14,8 @@ use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\Notification\Notification;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;

    @@ -59,7 +61,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, NotifierInterface $notifier, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -88,9 +90,15 @@ class ConferenceController extends AbstractController

                 $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

    +            $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));
    +
                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

    +        if ($form->isSubmitted()) {
    +            $notifier->send(new Notification('Can you check your submission? There are some problems with it.', ['browser']));
    +        }
    +
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

اعلان‌گر، *اعلان (notification)* را از طریق یک *کانال (channel)* به *گیرندگان (recipients)* *ارسال می‌کند*.

اعلان دارای یک عنوان، یک محتوای اختیاری و یک درجه‌ی اهمیت است.

یک اعلان با توجه به اهمیتش، به یک یا چند کانال ارسال می‌گردد. شما می‌توانید پیغام‌های ضروری و فوری را ازطریق پیامک و پیغام‌های معمولی را از طریق رایانامه ارسال کنید.

ما برای اعلان‌های مرورگر، گیرنده نداریم.

.. index::
    single: Twig;for

اعلان‌های مرورگر، از طریق بخش *اعلان*، از *پیغام‌های flash* استفاده می‌کند. ما نیاز داریم که آن‌ها را با به‌روزرسانی قالب کنفرانس، نمایش دهیم:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -3,6 +3,13 @@
     {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

     {% block body %}
    +    {% for message in app.flashes('notification') %}
    +        <div class="alert alert-info alert-dismissible fade show">
    +            {{ message }}
    +            <button type="button" class="close" data-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    +        </div>
    +    {% endfor %}
    +
         <h2 class="mb-5">
             {{ conference }} Conference
         </h2>

حالا به کاربران اطلاع داده می‌شود که کامنت‌های ارسالی‌شان تعدیل می‌گردد:

.. figure:: screenshots/form-success-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

به عنوان یک دست‌آورد جانبی، اگر خطایی در فرم وجود داشته باشد، ما یک پیغام اعلان زیبا در بالای وب‌سایت دریافت می‌کنیم:

.. figure:: screenshots/form-error-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. tip::

    پیغام‌های Flash، از سیستم *نشست HTTP* به عنوان انبار میانی استفاده می‌کنند. اثر اصلی این مسئله این است که نهان‌سازی HTTP در غیرفعال می‌شود چرا که سیستم نشست (session) باید شروع شده باشد تا بتوانیم ببینیم پیغام یا پیغام‌هایی وجود دارد یا خیر.

    این دلیل آن است که ما قطعه‌ی مربوط به پیغام‌های flash را در قالب ``show.html.twig`` اضافه کردیم و آن را در قالب پایه قرار ندادیم. چرا که اگر این کار را می‌کردیم، نهان‌سازی HTTP را در صفحه‌ی اصلی از دست می‌دادیم.

اطلاع‌رسانی به مدیران از طریق رایانامه
------------------------------------------------------------------------

به جای ارسال رایانامه از طریق ``MailerInterface`` برای اطلاع‌رسانی به مدیر در مورد ارسال‌شدن یک کامنت، در رسیدگی‌کننده‌ی پیغام از کامپوننت اعلانگر استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -4,14 +4,14 @@ namespace App\MessageHandler;

     use App\ImageOptimizer;
     use App\Message\CommentMessage;
    +use App\Notification\CommentReviewNotification;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    -use Symfony\Bridge\Twig\Mime\NotificationEmail;
    -use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Handler\MessageHandlerInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Workflow\WorkflowInterface;

     class CommentMessageHandler implements MessageHandlerInterface
    @@ -21,22 +21,20 @@ class CommentMessageHandler implements MessageHandlerInterface
         private $commentRepository;
         private $bus;
         private $workflow;
    -    private $mailer;
    +    private $notifier;
         private $imageOptimizer;
    -    private $adminEmail;
         private $photoDir;
         private $logger;

    -    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, ImageOptimizer $imageOptimizer, string $adminEmail, string $photoDir, LoggerInterface $logger = null)
    +    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, NotifierInterface $notifier, ImageOptimizer $imageOptimizer, string $photoDir, LoggerInterface $logger = null)
         {
             $this->entityManager = $entityManager;
             $this->spamChecker = $spamChecker;
             $this->commentRepository = $commentRepository;
             $this->bus = $bus;
             $this->workflow = $commentStateMachine;
    -        $this->mailer = $mailer;
    +        $this->notifier = $notifier;
             $this->imageOptimizer = $imageOptimizer;
    -        $this->adminEmail = $adminEmail;
             $this->photoDir = $photoDir;
             $this->logger = $logger;
         }
    @@ -62,13 +60,7 @@ class CommentMessageHandler implements MessageHandlerInterface

                 $this->bus->dispatch($message);
             } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    -            $this->mailer->send((new NotificationEmail())
    -                ->subject('New comment posted')
    -                ->htmlTemplate('emails/comment_notification.html.twig')
    -                ->from($this->adminEmail)
    -                ->to($this->adminEmail)
    -                ->context(['comment' => $comment])
    -            );
    +            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
             } elseif ($this->workflow->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

متد ``getAdminRecipients()``، مدیران گیرنده را همانطور که در پیکربندی اعلانگر پیکربندی شده است، بازمی‌گرداند؛ آن را به‌روزرسانی کرده و آدرس رایانامه‌تان را به آن بیافزایید:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/notifier.yaml
    +++ b/config/packages/notifier.yaml
    @@ -13,4 +13,4 @@ framework:
                 medium: ['email']
                 low: ['email']
             admin_recipients:
    -            - { email: admin@example.com }
    +            - { email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%" }

حالا کلاس ``CommentReviewNotification`` را ایجاد کنید:

.. code-block:: php
    :caption: src/Notification/CommentReviewNotification.php

    namespace App\Notification;

    use App\Entity\Comment;
    use Symfony\Component\Notifier\Message\EmailMessage;
    use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
    use Symfony\Component\Notifier\Notification\Notification;
    use Symfony\Component\Notifier\Recipient\Recipient;

    class CommentReviewNotification extends Notification implements EmailNotificationInterface
    {
        private $comment;

        public function __construct(Comment $comment)
        {
            $this->comment = $comment;

            parent::__construct('New comment posted');
        }

        public function asEmailMessage(Recipient $recipient, string $transport = null): ?EmailMessage
        {
            $message = EmailMessage::fromNotification($this, $recipient, $transport);
            $message->getMessage()
                ->htmlTemplate('emails/comment_notification.html.twig')
                ->context(['comment' => $this->comment])
            ;

            return $message;
        }
    }

متد ``asEmailMessage()`` در رابط ``EmailNotificationInterface`` اختیاری است، اما این امکان را می‌دهد تا رایانامه را سفارشی‌سازی کنید.

یک مزیت استفاده از اعلانگر به جای ارسال مستقیم رایانامه از طریق mailer این است که اعلان را از «کانال» مورد استفاده برای آن مجزا می‌کند. همانطور که می‌توانید ببینید، هیچ چیزی صریحاً نمی‌گوید که اعلان باید از طریق رایانامه ارسال گردد.

در عوض، کانال در ``config/packages/notifier.yaml`` بر اساس *درجه‌ی اهمیت* اعلان (به صورت پبشفرض ``low``)، پیکربندی شده است:

.. code-block:: yaml
    :caption: config/packages/notifier.yaml
    :class: ignore

    framework:
    notifier:
        channel_policy:
            # use chat/slack, chat/telegram, sms/twilio or sms/nexmo
            urgent: ['email']
            high: ['email']
            medium: ['email']
            low: ['email']

ما در مورد کانال‌های ``browser`` و ``email`` صحبت کردیم. بیایید چند کانال جالب‌تر ببینیم.

چت‌کردن با مدیران
---------------------------------

.. index::
    single: Slack

بیایید صادق باشیم، ما منتظر بازخوردهای مثبت هستیم. یا حداقل بازخوردی سازنده. اگر کسی کامنتی حاوی کلماتی همچون «عالی» یا «فوق‌العاده» ارسال کند، ما ممکن است بخواهیم آن را زودتر از سایر کامنت‌ها بپذیریم.

ما می‌خواهیم برای چنین پیغام‌هایی، علاوه بر رایانامه‌های همیشگی، بر روی سیستم‌های پیغام‌رسانی آنی مثل Slack یا Telegram، هشدار دریافت کنیم.

.. index::
    single: Components;Notifier
    single: Notifier

پشتیبانی از Slack را بر روی اعلانگر سیمفونی نصب کنید:

.. code-block:: bash

    $ symfony composer req slack-notifier

برای شروع، DSN مربوط به Slack را با یک توکن دسترسی Slack و شناسه‌ی کانال Slack‌ای که می‌خواهید به آن پیغام‌ها را ارسال نمایید، بسازید: ``slack://ACCESS_TOKEN@default?channel=CHANNEL``.

.. index::
    single: Command;secrets:set

از آنجایی که توکن دسترسی، حساس است، DSN مربوط به Slack را در انبار رمز ذخیره کنید:

.. code-block:: bash
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN

همین کار را برای محیط عمل‌آوری نیز انجام دهید:

.. code-block:: bash
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ APP_ENV=prod symfony console secrets:set SLACK_DSN

پشتیبانی از Slack را فعال کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/notifier.yaml
    +++ b/config/packages/notifier.yaml
    @@ -1,7 +1,7 @@
     framework:
         notifier:
    -        #chatter_transports:
    -        #    slack: '%env(SLACK_DSN)%'
    +        chatter_transports:
    +            slack: '%env(SLACK_DSN)%'
             #    telegram: '%env(TELEGRAM_DSN)%'
             #texter_transports:
             #    twilio: '%env(TWILIO_DSN)%'

کلاس اعلان را به‌روزرسانی کنید تا پیغام‌ها را با توجه به محتوای متن آن‌ها راه‌یابی کند (یک regex ساده این کار را انجام می‌دهد):

.. code-block:: diff
    :caption: patch_file

    --- a/src/Notification/CommentReviewNotification.php
    +++ b/src/Notification/CommentReviewNotification.php
    @@ -29,4 +29,15 @@ class CommentReviewNotification extends Notification implements EmailNotificatio

             return $message;
         }
    +
    +    public function getChannels(Recipient $recipient): array
    +    {
    +        if (preg_match('{\b(great|awesome)\b}i', $this->comment->getText())) {
    +            return ['email', 'chat/slack'];
    +        }
    +
    +        $this->importance(Notification::IMPORTANCE_LOW);
    +
    +        return ['email'];
    +    }
     }

ما همچنین درجه‌ی اهمیت کامنت‌های «معمولی» را تغییر دادیم تا کمی طراحی رایانامه را اصلاح کنیم.

و تمام! یک کامنت که در متن آن «awesome» وجود دارد را ارسال کنید، شما باید یک پیغام در Slack دریافت کنید.

همچون رایانامه‌ها، اینجا هم می‌توانید ``ChatNotificationInterface`` را پیاده‌سازی کنید تا نحوه‌ی renderشدن پیشفرض پیغام Slack را بازنویسی کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Notification/CommentReviewNotification.php
    +++ b/src/Notification/CommentReviewNotification.php
    @@ -3,12 +3,17 @@
     namespace App\Notification;

     use App\Entity\Comment;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
    +use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
    +use Symfony\Component\Notifier\Message\ChatMessage;
     use Symfony\Component\Notifier\Message\EmailMessage;
    +use Symfony\Component\Notifier\Notification\ChatNotificationInterface;
     use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\Recipient\Recipient;

    -class CommentReviewNotification extends Notification implements EmailNotificationInterface
    +class CommentReviewNotification extends Notification implements EmailNotificationInterface, ChatNotificationInterface
     {
         private $comment;

    @@ -30,6 +35,28 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
             return $message;
         }

    +    public function asChatMessage(Recipient $recipient, string $transport = null): ?ChatMessage
    +    {
    +        if ('slack' !== $transport) {
    +            return null;
    +        }
    +
    +        $message = ChatMessage::fromNotification($this, $recipient, $transport);
    +        $message->subject($this->getSubject());
    +        $message->options((new SlackOptions())
    +            ->iconEmoji('tada')
    +            ->iconUrl('https://guestbook.example.com')
    +            ->username('Guestbook')
    +            ->block((new SlackSectionBlock())->text($this->getSubject()))
    +            ->block(new SlackDividerBlock())
    +            ->block((new SlackSectionBlock())
    +                ->text(sprintf('%s (%s) says: %s', $this->comment->getAuthor(), $this->comment->getEmail(), $this->comment->getText()))
    +            )
    +        );
    +
    +        return $message;
    +    }
    +
         public function getChannels(Recipient $recipient): array
         {
             if (preg_match('{\b(great|awesome)\b}i', $this->comment->getText())) {

این بهتر است، اما بیایید یک گام جلوتر برویم. فوق‌العاده نبود اگر قادر بودیم که کامنت را مستقیماً از درون Slack تأیید یا رد کنیم؟

اعلان را تغییر دهیم تا URL بازبینی را بپذیرد و ۲ دکمه در پیغام Slack اضافه کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Notification/CommentReviewNotification.php
    +++ b/src/Notification/CommentReviewNotification.php
    @@ -3,6 +3,7 @@
     namespace App\Notification;

     use App\Entity\Comment;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackActionsBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
     use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
    @@ -16,10 +17,12 @@ use Symfony\Component\Notifier\Recipient\Recipient;
     class CommentReviewNotification extends Notification implements EmailNotificationInterface, ChatNotificationInterface
     {
         private $comment;
    +    private $reviewUrl;

    -    public function __construct(Comment $comment)
    +    public function __construct(Comment $comment, string $reviewUrl)
         {
             $this->comment = $comment;
    +        $this->reviewUrl = $reviewUrl;

             parent::__construct('New comment posted');
         }
    @@ -52,6 +55,10 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
                 ->block((new SlackSectionBlock())
                     ->text(sprintf('%s (%s) says: %s', $this->comment->getAuthor(), $this->comment->getEmail(), $this->comment->getText()))
                 )
    +            ->block((new SlackActionsBlock())
    +                ->button('Accept', $this->reviewUrl, 'primary')
    +                ->button('Reject', $this->reviewUrl.'?reject=1', 'danger')
    +            )
             );

             return $message;

حالا نوبت دنبال کردن تغییرات به صورت عقب‌رو است، اول رسیدگی‌کننده به پیغام را بهٰ‌روزرسانی کنید تا URL بازبینی را بدهد:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -60,7 +60,8 @@ class CommentMessageHandler implements MessageHandlerInterface

                 $this->bus->dispatch($message);
             } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    -            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
    +            $notification = new CommentReviewNotification($comment, $message->getReviewUrl());
    +            $this->notifier->send($notification, ...$this->notifier->getAdminRecipients());
             } elseif ($this->workflow->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

همانطور که می‌توانید ببینید، URL بازبینی باید بخشی از پیغام کامنت باشد، حالا بیایید آن را اضافه کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Message/CommentMessage.php
    +++ b/src/Message/CommentMessage.php
    @@ -5,14 +5,21 @@ namespace App\Message;
     class CommentMessage
     {
         private $id;
    +    private $reviewUrl;
         private $context;

    -    public function __construct(int $id, array $context = [])
    +    public function __construct(int $id, string $reviewUrl, array $context = [])
         {
             $this->id = $id;
    +        $this->reviewUrl = $reviewUrl;
             $this->context = $context;
         }

    +    public function getReviewUrl(): string
    +    {
    +        return $this->reviewUrl;
    +    }
    +
         public function getId(): int
         {
             return $this->id;

در نهایت، کنترلرها را به‌روزرسانی کنید تا URL بازبینی را تولید کرده و آن را به سازنده‌ی پیغام کامنت بدهند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -12,6 +12,7 @@ use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    @@ -51,7 +52,8 @@ class AdminController extends AbstractController
             $this->entityManager->flush();

             if ($accepted) {
    -            $this->bus->dispatch(new CommentMessage($comment->getId()));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl));
             }

             return $this->render('admin/review.html.twig', [
    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -17,6 +17,7 @@ use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Annotation\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
     use Twig\Environment;

     class ConferenceController extends AbstractController
    @@ -88,7 +89,8 @@ class ConferenceController extends AbstractController
                     'permalink' => $request->getUri(),
                 ];

    -            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl, $context));

                 $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));

جداسازی (decoupling) کد باعث لزوم انجام تغییرات در جاهای بیشتری است، اما آزمودن، دلیل‌‌آوری و بازاستفاده را آسان‌تر می‌کند.

مجدداً تلاش کنید، حلا باید پیغام در شکل مناسبی باشد:

.. image:: images/slack-message.png
    :align: center

ناهمزمان‌کردن تمام موارد
-----------------------------------------------

بگذارید مشکلی جزئی که باید آن را تصحیح کنیم را توضیح دهد. برای هر کامنت، ما یک رایانامه و یک پیغام Slack دریافت می‌کنیم. اگر خطایی در پیغام Slack رخ دهد (شناسه کانال اشتباه، توکن غلط و ...)، پیغام‌رسان قبل از دورانداختن پیغام، ۳ بار بازتلاش انجام می‌دهد. اما از آنجایی که رایانامه در همان بار اول ارسال شده است، ما ۳ بار رایانامه دریافت می‌کنیم و پیغام Slack را نیز دریافت نخواهیم کرد. یک راه برای حل این مشکل این است که همانند رایانامه‌ها، پیغام‌های Slack را نیز به صورت ناهمزمان ارسال کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -20,3 +20,5 @@ framework:
                 # Route your messages to the transports
                 App\Message\CommentMessage: async
                 Symfony\Component\Mailer\Messenger\SendEmailMessage: async
    +            Symfony\Component\Notifier\Message\ChatMessage: async
    +            Symfony\Component\Notifier\Message\SmsMessage: async

از آنجایی که همه‌چیز ناهمزمان است، پیغام‌ها از هم مستقل می‌شوند. ما همچنین پیغام‌های ناهمزمان پیامک را هم فعال کرده‌ایم که اگر خواستید بر روی تلفن‌همراه‌تان هم اطلاع‌رسانی شوید.

اطلاع‌رسانی به کاربران از طریق رایانامه
--------------------------------------------------------------------------

آخرین وظیفه اطلاع‌رسانی به کاربران در هنگامی است که کامنت‌شان تأیید گردیده است. چطور است که پیاده‌سازی این مورد به خودتان واگذار شود؟

.. sidebar:: بیشتر بدانید

    * `پیغام‌های flash در سیمفونی <https://symfony.com/doc/current/controller.html#flash-messages>`_.
