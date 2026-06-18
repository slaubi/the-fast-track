Benachrichtigungen auf allen Kanälen
=====================================

Die Gästebuchanwendung sammelt Feedback zu den Konferenzen. Wir sind jedoch nicht gut darin, unseren Nutzer*innen Feedback zu geben.

Da Kommentare moderiert werden, verstehen sie wahrscheinlich nicht, warum ihre Kommentare nicht sofort veröffentlicht werden. Sie könnten sie sogar erneut einreichen, weil sie denken, dass es technische Probleme gab. Ihnen Feedback zu geben, nachdem sie einen Kommentar geschrieben haben, wäre toll.

Außerdem sollten wir sie wahrscheinlich informieren, sobald ihr Kommentar veröffentlicht wurde. Wir verlangen ihre E-Mail-Adresse, also sollten wir sie auch verwenden.

Es gibt viele Möglichkeiten, Nutzer*innen zu benachrichtigen. E-Mail ist das Erste, woran Du vielleicht denkst. Benachrichtigungen innerhalb der Webanwendung sind ein weitere Möglichkeit. Wir könnten sogar überlegen, SMS-Nachrichten zu versenden und eine Nachricht auf Slack oder Telegram zu posten. Es gibt viele Möglichkeiten.

.. index::
    single: Components;Notifier
    single: Notifier

Die Symfony Notifier-Komponente implementiert viele Benachrichtigungsstrategien.

Benachrichtigungen von Webanwendungen im Browser senden
-------------------------------------------------------

.. index::
    single: Flash Messages

Lass uns in einem ersten Schritt die Nutzer*innen direkt im Browser, nachdem sie einen Kommentar abgegeben haben darüber informieren, dass Kommentare moderiert werden:

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
    @@ -45,7 +47,8 @@ final class ConferenceController extends AbstractController
             Request $request,
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

Der Notifier *sendet* (sends) eine *Nachricht* (notification) an die *Empfänger* (recipients) über einen *Kanal* (channel).

Eine Benachrichtigung hat einen Betreff (subject), einen optionalen Inhalt (content) und eine Wichtigkeit (importance).

Je nach Wichtigkeit wird eine Benachrichtigung auf einem oder mehreren Kanälen gesendet. Du kannst beispielsweise dringende Benachrichtigungen per SMS und regelmäßige Benachrichtigungen per E-Mail versenden.

Für Browser-Benachrichtigungen haben wir keine Empfänger.

.. index::
    single: Twig;for

Die Browser-Benachrichtigung verwendet *Flash-Messages* über den *Benachrichtigungsbereich*. Wir können sie anzeigen, indem wir das Konferenz-Template anpassen:

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

Nutzer*innen werden nun darüber informiert, dass ihr Kommentar moderiert wird:

.. figure:: screenshots/form-success-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Als zusätzlichen Bonus haben wir eine nette Benachrichtigung oben in der Website, wenn ein Formularfehler vorliegt:

.. figure:: screenshots/form-error-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. tip::

    Flash-Meldungen verwenden das *HTTP-Session*-System als Speichermedium. Die wichtigste Auswirkung ist, dass der HTTP-Cache deaktiviert ist, da das Session-System gestartet werden muss, um zu prüfen ob Nachrichten vorliegen.

    Dies ist der Grund, warum wir den Code für die Flash-Meldungen in das ``show.html.twig``-Template und nicht ins Basis-Template eingefügt haben. Sonst hätten wir den HTTP-Cache für die Homepage verloren.

Administrator*innen per E-Mail benachrichtigen
----------------------------------------------

Anstatt über das ``MailerInterface`` eine E-Mail zu senden, um die Administrator*innen über neue Kommentare zu informieren, wechseln wir zur Notifier-Komponente im Message-Handler:

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

Die ``getAdminRecipients()``-Methode gibt die Admin-Empfänger wie in der Notifier-Konfiguration konfiguriert zurück; aktualisiere sie jetzt, um Deine eigene E-Mail-Adresse hinzuzufügen:

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

Erstelle nun die ``CommentReviewNotification``-Klasse:

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

Die ``asEmailMessage()``-Methode des ``EmailNotificationInterface`` ist optional, aber sie erlaubt es, die E-Mail anzupassen.

Ein Vorteil der Verwendung des Notifiers anstelle direkter Verwendung des Mailers, um E-Mails zu versenden, besteht darin, dass Ersterer die Benachrichtigung von dem dafür verwendeten "Kanal" entkoppelt. Wie Du sehen kannst, steht da nicht explizit, dass die Benachrichtigung per E-Mail erfolgen soll.

Stattdessen wird der Kanal in ``config/packages/notifier.yaml`` abhängig von der *Wichtigkeit* der Benachrichtigung konfiguriert (Grundeinstellung ist ``low``):

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

Wir haben über die Kanäle ``browser`` und ``email`` gesprochen. Lass uns ein paar ausgefallenere anschauen.

Mit Administrator*innen chatten
-------------------------------

.. index::
    single: Slack

Seien wir ehrlich, wir alle sehnen uns nach positivem oder zumindest konstruktivem Feedback. Wenn jemand einen Kommentar mit Wörtern wie "toll" oder "awesome" schreibt, sollten wir diesen vielleicht schneller akzeptieren als andere.

Auf solche Kommentare möchten wir zusätzlich zur normalen E-Mail in einem Instant Messaging-System wie Slack oder Telegram aufmerksam gemacht werden.

.. index::
    single: Components;Notifier
    single: Notifier

Installiere den Slack-Support für Symfony Notifier:

.. code-block:: terminal

    $ symfony composer req slack-notifier

Erstelle den Slack-DSN mit einem Slack-Zugriffs-Token und dem Slack-Channel-Identifier, an den Du Nachrichten senden möchtest: ``slack://ACCESS_TOKEN@default?channel=CHANNEL``.

.. index::
    single: Command;secrets:set

Da das Zugriffs-Token eine sensible Information ist, speichere den Slack-DSN im Secret-Store:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN

Gleiches gilt für das Produktivsystem:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN --env=prod

Passe die Benachrichtigungsklasse an, um Nachrichten abhängig vom Kommentartext weiterzuleiten (ein einfacher regulärer Ausdruck erledigt diese Aufgabe):

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

Wir haben auch die Wichtigkeit der "normalen" Kommentare geändert, da dies das Design der E-Mail leicht verändert.

Und fertig! Schreibe einen Kommentar mit "awesome" im Text und erhalte eine Nachricht über Slack.

Wie bei E-Mails kannst Du ein ``ChatNotificationInterface`` implementieren, um die Standard-Darstellung der Slack-Nachricht zu überschreiben:

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

Schon besser, aber gehen wir noch einen Schritt weiter. Wäre es nicht toll, einen Kommentar direkt aus Slack akzeptieren oder ablehnen zu können?

Erweitere die Benachrichtigung, damit sie die Review-URL annimmt, und füge zwei Buttons zur Slack-Nachricht hinzu:

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

Es geht nun darum, Änderungen rückwärts zu verfolgen. Aktualisiere zunächst den Message-Handler, um die Review-URL zu übergeben:

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

Wie Du sehen kannst, sollte die Review-URL Teil der Kommentar-Nachricht sein, fügen wir sie jetzt hinzu:

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

Aktualisiere schließlich die Controller, um die Review-URL zu generieren und übergebe sie an den Constructor der Kommentar-Nachricht:

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

Code zu entkoppeln bedeutet Änderungen an mehreren Stellen, erleichtert aber das Testen, Wiederverwenden und Durchdenken unseres Codes.

Versuche es noch einmal, die Nachricht sollte jetzt in Ordnung sein:

.. image:: images/slack-message.png
    :align: center

Asynchron auf der ganzen Linie
------------------------------

Benachrichtigungen werden standardmäßig asynchron verschickt, genauso wie E-Mails:

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

Sollten wir asynchrone Nachrichten deaktivieren, würden wir ein kleines Problem haben. Für jeden Kommentar erhalten wir eine E-Mail und eine Slack-Nachricht. Wenn die Slack-Nachricht fehlerhaft ist (falsche Kanal-ID, falsches Token, ...), wird dreimal versucht die Messenger-Nachricht zu versenden, bevor sie verworfen wird. Da jedoch die E-Mail zuerst gesendet wird, erhalten wir drei E-Mails und keine Slack-Nachrichten.

Sobald alles asynchron ist, sind die Nachrichten voneinander unabhängig. SMS-Nachrichten sind bereits asynchron konfiguriert, falls Du auch auf Deinem Telefon benachrichtigt werden möchtest.

Nutzer*innen per E-Mail benachrichtigen
---------------------------------------

Die letzte Aufgabe besteht darin, die Nutzer*innen zu benachrichtigen, wenn deren Kommentar genehmigt wird. Wie wäre es, wenn Du das selbst umsetzt?

.. sidebar:: Weiterführendes

    * `Symfony Flash-Meldungen`_.

.. _`Symfony Flash-Meldungen`: https://symfony.com/doc/current/controller.html#flash-messages
