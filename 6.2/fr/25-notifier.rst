Notifier à tout prix
=====================

L'application Livre d'or recueille les retours sur les conférences. Mais nous, de notre côté, quels retours donnons-nous aux internautes ?

Comme les commentaires sont modérés, il leur est difficile de comprendre pourquoi ils ne sont pas publiés instantanément. Ils pourraient même avoir envie de les soumettre à nouveau, en pensant qu'il y a eu des problèmes techniques. Ce serait génial de donner un retour après avoir posté un commentaire.

De plus, nous devrions probablement les notifier lorsque leurs commentaires auront été publiés. Nous leur demandons leur adresse email, alors autant l'utiliser.

Il y a plusieurs façons d'envoyer des notifications. L'email est le premier support auquel vous pouvez penser, mais les notifications dans l'application web sont une autre possibilité. On pourrait même penser à envoyer des SMS ou à poster un message sur Slack ou Telegram. Les options sont nombreuses.

.. index::
    single: Components;Notifier
    single: Notifier

Le composant *Symfony Notifier* propose de nombreuses stratégies de notification :

Envoyez des notifications d'application web dans le navigateur
--------------------------------------------------------------

.. index::
    single: Flash Messages

Dans un premier temps, avertissons les internautes que les commentaires sont modérés directement dans le navigateur après leur soumission :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -15,6 +15,8 @@ use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\Notification\Notification;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
    @@ -46,6 +48,7 @@ class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        NotifierInterface $notifier,
             #[Autowire('%photo_dir%')] string $photoDir,
         ): Response {
             $comment = new Comment();
    @@ -74,9 +77,15 @@ class ConferenceController extends AbstractController
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
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

Le notifier *envoie* une *notification* aux *destinataires* via un *canal*.

Une notification a un sujet, un contenu facultatif et une importance.

Une notification est envoyée sur un ou plusieurs canaux en fonction de son importance. Vous pouvez envoyer des notifications urgentes par SMS et des notifications régulières par email par exemple.

Pour les notifications par navigateur, nous n'avons pas de destinataires.

.. index::
    single: Twig;for

La notification par navigateur utilise des *messages flash* via la section *notification*. Nous devons les afficher en mettant à jour le template d'une conférence :

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
    +            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    +        </div>
    +    {% endfor %}
    +
         <h2 class="mb-5">
             {{ conference }} Conference
         </h2>

Les internautes seront maintenant prévenus que leurs commentaires sont modérés :

.. figure:: screenshots/form-success-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

En prime, nous avons une belle notification en haut du site s'il y a une erreur de formulaire :

.. figure:: screenshots/form-error-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. tip::

    Les messages flash utilisent le système de *session HTTP* comme support de stockage. La principale conséquence est que le cache HTTP est désactivé, car le système de session doit être démarré pour vérifier les messages.

    C'est la raison pour laquelle nous avons ajouté le code pour les messages flash dans le template ``show.html.twig`` et non dans le template de base, car nous aurions perdu le cache HTTP pour la page d'accueil.

Notifier les admins par email
-----------------------------

Au lieu d'envoyer un email via ``MailerInterface`` pour avertir l'admin qu'un commentaire vient d'être posté, utilisez le composant Notifier dans le gestionnaire de messages :

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
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

La méthode ``getAdminRecipients()`` retourne les destinataires admin tels que configurés dans la configuration du notifier ; mettez-la à jour maintenant pour ajouter votre propre adresse email :

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

Maintenant, créez la classe ``CommentReviewNotification`` :

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

La méthode ``asEmailMessage()`` de ``EmailNotificationInterface`` est facultative, mais elle permet de personnaliser l'email.

L'un des avantages d'utiliser le Notifier plutôt que d'expédier l'email directement est que cela dissocie la notification du "canal" utilisé pour l'envoyer. Comme vous pouvez le constater, rien n'indique explicitement que la notification doit être envoyée par email.

Au lieu de cela, le canal est configuré dans ``config/packages/notifier.yaml`` en fonction de l'*importance* de la notification (``low`` par défaut) :

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

Nous avons parlé des canaux ``email`` et ``browser``. Voyons maintenant des cas plus sympas.

Discuter avec les admins
------------------------

.. index::
    single: Slack

Soyons honnêtes, nous attendons tous des commentaires positifs. Ou au moins un retour constructif. Si quelqu'un poste un commentaire avec des mots comme "fantastique" ou "génial", nous pourrions vouloir les accepter plus rapidement que les autres.

Pour de tels messages, nous voulons recevoir une alerte sur un système de messagerie instantanée comme Slack ou Telegram, en plus de l'email normal.

.. index::
    single: Components;Notifier
    single: Notifier

Installez la prise en charge de Slack pour Symfony Notifier :

.. code-block:: terminal

    $ symfony composer req slack-notifier

Pour commencer, ajouter au DSN Slack un jeton d'accès Slack et l'identifiant du canal Slack sur lequel vous voulez envoyer des messages : ``slack://ACCESS_TOKEN@default?channel=CHANNEL``.

.. index::
    single: Command;secrets:set

Comme le jeton d'accès est sensible, stockez le DSN Slack avec vos autres chaînes secrètes :

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN

Faites de même pour la production :

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN --env=prod

Activez le support du chatter Slack :

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

Mettez à jour la classe Notification pour router les messages en fonction du contenu du texte du commentaire (une simple expression régulière fera l'affaire) :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Notification/CommentReviewNotification.php
    +++ b/src/Notification/CommentReviewNotification.php
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

Nous avons également changé l'importance des commentaires "normaux", car ils modifient légèrement le design de l'email.

Et voilà, c'est fait ! Soumettez un commentaire avec "awesome" dans le texte : vous devriez recevoir un message sur Slack.

Comme pour l'email, vous pouvez implémenter ``ChatNotificationInterface`` pour remplacer le rendu par défaut du message Slack :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Notification/CommentReviewNotification.php
    +++ b/src/Notification/CommentReviewNotification.php
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

C'est mieux, mais allons plus loin. Ne serait-il pas génial de pouvoir accepter ou rejeter un commentaire directement depuis Slack ?

Modifiez la notification pour accepter l'URL de validation et ajoutez deux boutons dans le message Slack :

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

Nous devons maintenant gérer les changements dans le sens inverse. Tout d'abord, mettez à jour le gestionnaire de message pour passer l'URL de validation :

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
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

Comme vous pouvez le voir, l'URL de validation devrait faire partie du message de commentaire. Ajoutons-la maintenant :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Message/CommentMessage.php
    +++ b/src/Message/CommentMessage.php
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

Enfin, modifiez les contrôleurs pour générer l'URL de validation et passez-la dans le constructeur du message de commentaire :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -12,6 +12,7 @@ use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
     use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
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
    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -18,6 +18,7 @@ use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Annotation\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

     class ConferenceController extends AbstractController
     {
    @@ -75,7 +76,8 @@ class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl, $context));

                 $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));

Le découplage de code implique des changements dans un plus grand nombre d'endroits, mais il facilite les tests, le raisonnement et la réutilisation.

Essayez encore une fois, le message devrait avoir une bonne tête maintenant :

.. image:: images/slack-message.png
    :align: center

Utiliser le mode asynchrone
---------------------------

Les notifications sont envoyées de manière asynchrone par défaut, comme pour les emails :

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

Si nous désactivions l'envoi des messages en asynchrone, nous aurions un problème. Pour chaque commentaire, nous recevons un email et un message Slack. Si le message Slack génère une erreur (mauvais identifiant de canal, mauvais jeton, etc.), le message sera renvoyé trois fois avant d'être rejeté. Mais comme l'email est envoyé en premier, nous recevrons trois emails et aucun message Slack.

Dès que tout est asynchrone, les messages deviennent indépendants. Les messages SMS sont déjà configurés pour être envoyés de façon asynchrones au cas où vous souhaiteriez recevoir les notifications sur votre téléphone.

Notifier les internautes par email
----------------------------------

La dernière tâche consiste à notifier les internautes lorsque leur soumission est approuvée. Et si vous codiez cela vous même ?

.. sidebar:: Aller plus loin

    * `Messages flash Symfony`_.

.. _`Messages flash Symfony`: https://symfony.com/doc/current/controller.html#flash-messages
