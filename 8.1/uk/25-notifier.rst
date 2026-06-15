Сповіщення всілякими засобами
========================================================

Застосунок гостьової книги збирає відгуки про конференції. Але ми не найкращі в наданні зворотного зв'язку нашим користувачам.

Оскільки коментарі проходять перевірку, користувачі, ймовірно, не розуміють, чому їх коментарі не публікуються миттєво. Вони навіть можуть відправити їх повторно, думаючи, що є якісь технічні проблеми. Було б чудово надати їм зворотний зв'язок, після публікації коментаря.

Крім того, ми, мабуть, маємо сповістити їх, як тільки їх коментар було опубліковано. Ми просимо вказати їх адресу електронної пошти, тому нам краще використовувати її.

Існує безліч способів оповіщення користувачів. Електронна пошта — це перший засіб, про який ви можете подумати, але сповіщення у веб-застосунку — це ще один засіб. Ми могли б навіть подумати про відправку SMS-повідомлень, відправку повідомлення у Slack або Telegram. Є багато варіантів.

.. index::
    single: Components;Notifier
    single: Notifier

Компонент Symfony Notifier реалізує багато стратегій оповіщення.

Відправка сповіщень веб-застосунку в браузері
-------------------------------------------------------------------------------------

.. index::
    single: Flash Messages

У якості першого кроку сповістімо користувачів про те, що коментарі проходять перевірку — безпосередньо в браузері, після їх відправки:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -16,6 +16,8 @@ use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\Notification\Notification;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -45,8 +47,9 @@ final class ConferenceController extends AbstractController
             Request $request,
             #[MapEntity(mapping: ['slug' => 'slug'])]
             Conference $conference,
             CommentRepository $commentRepository,
    +        NotifierInterface $notifier,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -69,9 +72,15 @@ final class ConferenceController extends AbstractController
                 ];
                 $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

    +            $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));
    +
                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

    +        if ($form->isSubmitted()) {
    +            $notifier->send(new Notification('Can you check your submission? There are some problems with it.', ['browser']));
    +        }
    +
             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

Сповіщувач *відправляє* *сповіщення* до *отримувачів* за допомогою *каналу*

У сповіщення є тема, необов'язковий зміст і рівень важливості.

Сповіщення відправляється за допомогою одного або декількох каналів, залежно від рівня його важливості. Ви можете відправляти термінові сповіщення, наприклад, за допомогою SMS, а звичайні — за допомогою електронної пошти.

Для сповіщень браузера у нас немає одержувачів.

.. index::
    single: Twig;for

Сповіщення браузера використовує *миттєві повідомлення* за допомогою секції *notification*. Ми маємо відобразити їх, оновивши шаблон конференції:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -3,6 +3,13 @@
     {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

     {% block body %}
    +    {% for message in app.flashes('notification') %}
    +        <div class="alert alert-info alert-dismissible fade show">
    +            {{ message }}
    +            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    +        </div>
    +    {% endfor %}
    +
         <h2 class="mb-5">
             {{ conference }} Conference
         </h2>

Тепер користувачі будуть сповіщені про те, що їх подання проходить перевірку:

.. figure:: screenshots/form-success-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

У якості додаткового бонусу у нас є приємне сповіщення у верхній частині веб-сайту, якщо у формі є помилка:

.. figure:: screenshots/form-error-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. tip::

    Миттєві повідомлення використовують систему *HTTP-сесії* в якості носія даних. Головним наслідком цього є те, що HTTP-кеш  вимкнено, оскільки система сесій має бути запущена, щоб перевірити повідомлення.

    Саме з цієї причини ми додали фрагмент миттєвих повідомлень у шаблон ``show.html.twig``, а не в базовий, оскільки ми втратили б HTTP-кеш для головної сторінки.

Сповіщення адміністраторів за допомогою електронної пошти
-------------------------------------------------------------------------------------------------------------

Замість того щоб відправляти електронний лист за допомогою ``MailerInterface``, щоб сповістити адміністратора про те, що щойно опубліковано коментар, перейдіть до використання компонента Notifier в обробнику повідомлень:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -4,15 +4,15 @@ namespace App\MessageHandler;

     use App\ImageOptimizer;
     use App\Message\CommentMessage;
    +use App\Notification\CommentReviewNotification;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    -use Symfony\Bridge\Twig\Mime\NotificationEmail;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    -use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
    @@ -24,8 +24,7 @@ class CommentMessageHandler
             private CommentRepository $commentRepository,
             private MessageBusInterface $bus,
             private WorkflowInterface $commentStateMachine,
    -        private MailerInterface $mailer,
    -        #[Autowire('%admin_email%')] private string $adminEmail,
    +        private NotifierInterface $notifier,
             private ImageOptimizer $imageOptimizer,
             #[Autowire('%photo_dir%')] private string $photoDir,
             private ?LoggerInterface $logger = null,
    @@ -50,13 +49,7 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    -            $this->mailer->send((new NotificationEmail())
    -                ->subject('New comment posted')
    -                ->htmlTemplate('emails/comment_notification.html.twig')
    -                ->from($this->adminEmail)
    -                ->to($this->adminEmail)
    -                ->context(['comment' => $comment])
    -            );
    +            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
             } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

Метод ``getAdminRecipients()`` повертає список адміністраторів-одержувачів, що налаштований в конфігурації сповіщувача; оновіть його зараз, щоб додати власну адресу електронної пошти:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/notifier.yaml
    +++ w/config/packages/notifier.yaml
    @@ -9,4 +9,4 @@ framework:
                 medium: ['email']
                 low: ['email']
             admin_recipients:
    -            - { email: admin@example.com }
    +            - { email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%" }

Тепер створіть клас ``CommentReviewNotification``:

.. code-block:: php
    :caption: src/Notification/CommentReviewNotification.php

    namespace App\Notification;

    use App\Entity\Comment;
    use Symfony\Component\Notifier\Message\EmailMessage;
    use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
    use Symfony\Component\Notifier\Notification\Notification;
    use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;

    class CommentReviewNotification extends Notification implements EmailNotificationInterface
    {
        public function __construct(
            private Comment $comment,
        ) {
            parent::__construct('New comment posted');
        }

        public function asEmailMessage(EmailRecipientInterface $recipient, string $transport = null): ?EmailMessage
        {
            $message = EmailMessage::fromNotification($this, $recipient, $transport);
            $message->getMessage()
                ->htmlTemplate('emails/comment_notification.html.twig')
                ->context(['comment' => $this->comment])
            ;

            return $message;
        }
    }

Метод ``asEmailMessage()`` із ``EmailNotificationInterface`` є необов'язковим, але він дозволяє кастомізувати електронний лист.

Одна з переваг використання сповіщувача замість відправника безпосередньо, для відправки електронних листів, полягає в тому, що він відокремлює сповіщення від використовуваного для нього "каналу". Як ви можете бачити, ніщо явно не говорить про те, що сповіщення має бути відправлено за допомогою електронної пошти.

Натомість канал налаштовується в ``config/packages/notifier.yaml`` залежно від *рівня важливості* сповіщення (``низький`` за замовчуванням):

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

Ми говорили про канали ``браузер``  і ``електронна пошта``. Подивімося на більш незвичайні з них.

Чат з адміністраторами
------------------------------------------

.. index::
    single: Slack

Будьмо чесними, ми всі чекаємо позитивних відгуків. Або, принаймні, конструктивний зворотний зв'язок. Якщо хтось публікує коментар з такими словами, як "great" або "awesome", ми, можливо, захочемо прийняти його швидше, ніж інші.

Для таких повідомлень ми хочемо отримувати сповіщення в такій системі обміну миттєвими повідомленнями, як Slack або Telegram, на додаток до звичайного електронного листа.

.. index::
    single: Components;Notifier
    single: Notifier

Встановіть підтримку Slack для Symfony Notifier:

.. code-block:: terminal

    $ symfony composer req slack-notifier

Для початку скомпонуйте Slack DSN з токеном доступу Slack і ідентифікатором каналу Slack, куди ви хочете відправляти повідомлення: ``slack://ACCESS_TOKEN@default?channel=CHANNEL``.

.. index::
    single: Command;secrets:set

Оскільки токен доступу є чутливим, зберігайте Slack DSN у секретному сховищі:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN

Зробіть те ж саме для продакшн:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN --env=prod

Оновіть клас сповіщення, щоб маршрутизувати повідомлення залежно від вмісту тексту коментаря (простий регулярний вираз виконає цю роботу):

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -7,6 +7,7 @@ use Symfony\Component\Notifier\Message\EmailMessage;
     use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;
    +use Symfony\Component\Notifier\Recipient\RecipientInterface;

     class CommentReviewNotification extends Notification implements EmailNotificationInterface
     {
    @@ -26,4 +27,15 @@ class CommentReviewNotification extends Notification implements EmailNotificatio

             return $message;
         }
    +
    +    public function getChannels(RecipientInterface $recipient): array
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

Ми також змінили рівень важливості "звичайних" коментарів, оскільки він трохи змінює дизайн електронного листа.

І готово! Відправте коментар зі словом "awesome" в тексті, ви маєте отримати повідомлення у Slack.

Що стосується електронного листа, ви можете реалізувати ``ChatNotificationInterface``, щоб перевизначити візуалізацію повідомлення Slack за замовчуванням:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -3,13 +3,18 @@
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
     use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;
     use Symfony\Component\Notifier\Recipient\RecipientInterface;

    -class CommentReviewNotification extends Notification implements EmailNotificationInterface
    +class CommentReviewNotification extends Notification implements EmailNotificationInterface, ChatNotificationInterface
     {
         public function __construct(
             private Comment $comment,
    @@ -28,6 +33,28 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
             return $message;
         }

    +    public function asChatMessage(RecipientInterface $recipient, string $transport = null): ?ChatMessage
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
         public function getChannels(RecipientInterface $recipient): array
         {
             if (preg_match('{\b(great|awesome)\b}i', $this->comment->getText())) {

Краще, але зробімо ще один крок вперед. Хіба не було б чудово мати можливість прийняти або відхилити коментар безпосередньо зі Slack?

Змініть сповіщення, щоб воно приймало URL-адресу огляду, і додайте дві кнопки в повідомлення Slack:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -3,6 +3,7 @@
     namespace App\Notification;

     use App\Entity\Comment;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackActionsBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
     use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
    @@ -18,6 +19,7 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
     {
         public function __construct(
             private Comment $comment,
    +        private string $reviewUrl,
         ) {
             parent::__construct('New comment posted');
         }
    @@ -50,6 +52,10 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
                 ->block((new SlackSectionBlock())
                     ->text(sprintf('%s (%s) says: %s', $this->comment->getAuthor(), $this->comment->getEmail(), $this->comment->getText()))
                 )
    +            ->block((new SlackActionsBlock())
    +                ->button('Accept', $this->reviewUrl, 'primary')
    +                ->button('Reject', $this->reviewUrl.'?reject=1', 'danger')
    +            )
             );

             return $message;

Тепер справа за відстеженням змін у зворотному напрямку. По-перше, оновіть обробник повідомлень, щоб передати URL-адресу огляду:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -49,7 +49,8 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    -            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
    +            $notification = new CommentReviewNotification($comment, $message->getReviewUrl());
    +            $this->notifier->send($notification, ...$this->notifier->getAdminRecipients());
             } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

Як ви можете бачити, URL-адреса огляду має бути частиною повідомлення коментаря, додаймо його зараз:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Message/CommentMessage.php
    +++ w/src/Message/CommentMessage.php
    @@ -6,10 +6,16 @@ class CommentMessage
     {
         public function __construct(
             private int $id,
    +        private string $reviewUrl,
             private array $context = [],
         ) {
         }

    +    public function getReviewUrl(): string
    +    {
    +        return $this->reviewUrl;
    +    }
    +
         public function getId(): int
         {
             return $this->id;

Нарешті, оновіть контролери, щоб згенерувати URL-адресу огляду й передати його в конструктор повідомлення коментаря:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -12,6 +12,7 @@ use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
     use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    @@ -42,7 +43,8 @@ class AdminController extends AbstractController
             $this->entityManager->flush();

             if ($accepted) {
    -            $this->bus->dispatch(new CommentMessage($comment->getId()));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl));
             }

             return new Response($this->twig->render('admin/review.html.twig', [
    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -17,6 +17,7 @@ use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Attribute\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

     final class ConferenceController extends AbstractController
     {
    @@ -70,7 +71,8 @@ final class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl, $context));

                 $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));

Декомпозиція коду означає зміни в більшій кількості місць, але це полегшує тестування, аналіз і повторне використання.

Спробуйте ще раз, тепер повідомлення має бути у відповідному вигляді:

.. image:: images/slack-message.png
    :align: center

Перехід до асинхронності у всіх напрямках
-----------------------------------------------------------------------------

Сповіщення, як-от електронні листи, за замовчуванням надсилаються асинхронно.

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 5,6
    :class: ignore

    framework:
        messenger:
            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async

                # Route your messages to the transports
                App\Message\CommentMessage: async

Якби ми вимкнули асинхронні повідомлення, у нас виникла б невелика проблема. Для кожного коментаря ми отримуємо повідомлення електронної пошти й повідомлення Slack. Якщо в повідомленні Slack є помилки (неправильний ідентифікатор каналу, неправильний токен, ...), Messenger буде відправляти повідомлення повторно три рази, перш ніж відкине його. Але оскільки повідомлення електронної пошти відправляється першим, ми отримаємо 3 повідомлення електронної пошти й жодних повідомлень Slack.

Як тільки все стає асинхронним, повідомлення стають незалежними. SMS-повідомлення вже налаштовані як асинхронні на той випадок, якщо ви також хочете отримувати сповіщення на ваш телефон.

Сповіщення користувачів за допомогою електронної пошти
-------------------------------------------------------------------------------------------------------

Останнє завдання — повідомити користувачів, коли їх подання буде схвалено. А як щодо того, щоб дозволити вам реалізувати це самостійно?

.. sidebar:: Йдемо далі

    * `Миттєві повідомлення Symfony`_.

.. _`Миттєві повідомлення Symfony`: https://symfony.com/doc/current/controller.html#flash-messages
