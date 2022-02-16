Admins e-mailen
===============

.. index::
    single: Components;Mailer
    single: Mailer
    single: Emails

Om kwalitatief hoogwaardige feedback te garanderen, moet de admin alle reacties controleren. Wanneer een reactie in de ``ham`` of ``potential_spam`` state is, moet een *e-mail* worden gestuurd naar de admin met twee links: één om de opmerking te accepteren, en één om het te verwerpen.

Een e-mail instellen voor de admin
----------------------------------

Om het admin-e-mailadres op te slaan, gebruik je een containerparameter. Voor demonstratiedoeleinden laten we het ook toe om dit in te stellen via een omgevingsvariabele (zou niet nodig moeten zijn in het "echte leven"). Om de injectie van het admin e-mailadres in services te vergemakkelijken, kun je een container ``bind`` instelling definiëren:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -4,6 +4,7 @@
     # Put parameters here that don't need to change on each machine where the app is deployed
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
    +    default_admin_email: admin@example.com

     services:
         # default configuration for services in *this* file
    @@ -13,6 +14,7 @@ services:
             bind:
                 string $photoDir: "%kernel.project_dir%/public/uploads/photos"
                 string $akismetKey: "%env(AKISMET_KEY)%"
    +            string $adminEmail: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

Een omgevingsvariabele kan "verwerkt" worden voordat deze wordt gebruikt. Hier gebruiken we de ``default`` processor om terug te vallen op de waarde van de ``default_admin_email`` parameter als de ``ADMIN_EMAIL`` omgevingsvariabele niet bestaat.

Een notificatie-e-mail verzenden
--------------------------------

Om een e-mail te versturen, kan je kiezen tussen verschillende ``Email`` class abstracties; van ``Message``, het laagste niveau, tot ``NotificationEmail``, het hoogste niveau. Je zal waarschijnlijk het meest gebruik maken van de ``Email`` class, maar ``NotificationEmail`` is de perfecte keuze voor interne e-mailberichten.

Laten we de automatische validatie logica vervangen in de message handler:

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -7,6 +7,8 @@ use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    +use Symfony\Bridge\Twig\Mime\NotificationEmail;
    +use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Handler\MessageHandlerInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -18,15 +20,19 @@ class CommentMessageHandler implements MessageHandlerInterface
         private $commentRepository;
         private $bus;
         private $workflow;
    +    private $mailer;
    +    private $adminEmail;
         private $logger;

    -    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, LoggerInterface $logger = null)
    +    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, string $adminEmail, LoggerInterface $logger = null)
         {
             $this->entityManager = $entityManager;
             $this->spamChecker = $spamChecker;
             $this->commentRepository = $commentRepository;
             $this->bus = $bus;
             $this->workflow = $commentStateMachine;
    +        $this->mailer = $mailer;
    +        $this->adminEmail = $adminEmail;
             $this->logger = $logger;
         }

    @@ -51,8 +57,13 @@ class CommentMessageHandler implements MessageHandlerInterface

                 $this->bus->dispatch($message);
             } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    -            $this->workflow->apply($comment, $this->workflow->can($comment, 'publish') ? 'publish' : 'publish_ham');
    -            $this->entityManager->flush();
    +            $this->mailer->send((new NotificationEmail())
    +                ->subject('New comment posted')
    +                ->htmlTemplate('emails/comment_notification.html.twig')
    +                ->from($this->adminEmail)
    +                ->to($this->adminEmail)
    +                ->context(['comment' => $comment])
    +            );
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

De ``MailerInterface`` is het belangrijkste toegangspunt en maakt het mogelijk om e-mailberichten te versturen, middels de ``send()`` functie.

Om een e-mail te versturen, hebben we een afzender nodig (de ``From`` / ``Sender`` header). In plaats van dit expliciet in te stellen op de e-mail instantie, definieer je het globaal:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/mailer.yaml
    +++ b/config/packages/mailer.yaml
    @@ -1,3 +1,5 @@
     framework:
         mailer:
             dsn: '%env(MAILER_DSN)%'
    +        envelope:
    +            sender: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"

Het uitbreiden van de notificatie-e-mail template
-------------------------------------------------

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;url

De notificatie e-mailtemplate erft over van de standaard notificatie-e-mailtemplate die met Symfony wordt geleverd:

.. code-block:: html+twig
    :caption: templates/emails/comment_notification.html.twig

    {% extends '@email/default/notification/body.html.twig' %}

    {% block content %}
        Author: {{ comment.author }}<br />
        Email: {{ comment.email }}<br />
        State: {{ comment.state }}<br />

        <p>
            {{ comment.text }}
        </p>
    {% endblock %}

    {% block action %}
        <spacer size="16"></spacer>
        <button href="{{ url('review_comment', { id: comment.id }) }}">Accept</button>
        <button href="{{ url('review_comment', { id: comment.id, reject: true }) }}">Reject</button>
    {% endblock %}

De template overschrijft een paar blokken om de inhoud van de e-mail aan te passen en om een aantal links toe te voegen die de beheerder in staat stelt om een opmerking te accepteren of te weigeren. Elk route-argument dat geen geldige routeparameter is, wordt toegevoegd als een query string parameter (de URL om af te wijzen ziet er dan als volgt uit ``/admin/comment/review/42?reject=true`` ).

De standaard ``NotificationEmail`` template gebruikt `Inky`_ in plaats van HTML om e-mailberichten op te bouwen. Dit helpt bij het creëren van responsive e-mailberichten die ondersteund worden door populaire e-mailclients.

Voor een maximale compatibiliteit met e-mailreaders, voegt de notificatie-basislayout standaard alle stylesheets (via het CSS inliner package) inline toe.

Deze twee hulpmiddelen maken deel uit van optionele Twig extensies die moeten worden geïnstalleerd:

.. code-block:: terminal

    $ symfony composer req "twig/cssinliner-extra:^3" "twig/inky-extra:^3"

Absolute URL's genereren in een command
---------------------------------------

.. index::
    single: Twig;Link
    single: Link

Genereer jouw URL's in e-mailberichten met ``url()`` in plaats van ``path()``, omdat je daar absolute URL's nodig hebt (met protocol en host).

De e-mail wordt verzonden door de message handler, in een consolecontext. Het genereren van absolute URL's in een webcontext is gemakkelijker omdat we het protocol en het domein van de huidige pagina kennen. Dit is niet het geval in de context van een console.

Definieer het protocol en de domeinnaam dan ook expliciet:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -5,6 +5,11 @@
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
         default_admin_email: admin@example.com
    +    default_domain: '127.0.0.1'
    +    default_scheme: 'http'
    +
    +    router.request_context.host: '%env(default:default_domain:SYMFONY_DEFAULT_ROUTE_HOST)%'
    +    router.request_context.scheme: '%env(default:default_scheme:SYMFONY_DEFAULT_ROUTE_SCHEME)%'

     services:
         # default configuration for services in *this* file

De ``SYMFONY_DEFAULT_ROUTE_HOST`` en ``SYMFONY_DEFAULT_ROUTE_PORT`` omgevingsvariabelen worden lokaal automatisch ingesteld bij gebruik van de ``symfony`` CLI en bepaald op basis van de configuratie op Platform.sh.

Een route aan een controller verbinden
--------------------------------------

De ``review_comment`` route bestaat nog niet, laten we een admin controller aanmaken om dit af te handelen:

.. code-block:: php
    :caption: src/Controller/AdminController.php

    namespace App\Controller;

    use App\Entity\Comment;
    use App\Message\CommentMessage;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Messenger\MessageBusInterface;
    use Symfony\Component\Routing\Annotation\Route;
    use Symfony\Component\Workflow\Registry;
    use Twig\Environment;

    class AdminController extends AbstractController
    {
        private $twig;
        private $entityManager;
        private $bus;

        public function __construct(Environment $twig, EntityManagerInterface $entityManager, MessageBusInterface $bus)
        {
            $this->twig = $twig;
            $this->entityManager = $entityManager;
            $this->bus = $bus;
        }

        #[Route('/admin/comment/review/{id}', name: 'review_comment')]
        public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
        {
            $accepted = !$request->query->get('reject');

            $machine = $registry->get($comment);
            if ($machine->can($comment, 'publish')) {
                $transition = $accepted ? 'publish' : 'reject';
            } elseif ($machine->can($comment, 'publish_ham')) {
                $transition = $accepted ? 'publish_ham' : 'reject_ham';
            } else {
                return new Response('Comment already reviewed or not in the right state.');
            }

            $machine->apply($comment, $transition);
            $this->entityManager->flush();

            if ($accepted) {
                $this->bus->dispatch(new CommentMessage($comment->getId()));
            }

            return new Response($this->twig->render('admin/review.html.twig', [
                'transition' => $transition,
                'comment' => $comment,
            ]));
        }
    }

De URL om te reageren op een review begint met ``/admin/`` zodat deze meteen beschermd wordt met de firewall die we in de vorige stap gedefinieerd hebben. De admin moet geverifieerd zijn om toegang te krijgen tot deze actie.

In plaats van een ``Response`` instantie aan te maken, hebben we de verkorte methode ``render()`` gebruikt, die door de basis class van de ``AbstractController`` controller wordt voorzien.

.. index::
    single: Twig;extends
    single: Twig;block

Wanneer de review is afgerond, wordt een simpele template gebruikt om de admin te bedanken voor het harde werk:

.. code-block:: html+twig
    :caption: templates/admin/review.html.twig

    {% extends 'base.html.twig' %}

    {% block body %}
        <h2>Comment reviewed, thank you!</h2>

        <p>Applied transition: <strong>{{ transition }}</strong></p>
        <p>New state: <strong>{{ comment.state }}</strong></p>
    {% endblock %}

Een mailcatcher gebruiken
-------------------------

.. index::
    single: Docker;Mail Catcher

In plaats van een "echte" SMTP-server of een service van een derde partij te gebruiken om e-mailberichten te versturen, gebruiken we een mailcatcher. Een mailcatcher biedt een SMTP-server die de e-mailberichten niet aflevert, maar ze beschikbaar stelt via een webinterface. Gelukkig heeft Symfony reeds automatisch zo'n mailcatcher voor ons geconfigureerd:

.. code-block:: yaml
    :caption: docker-compose.override.yml
    :class: ignore

    services:
    ###> symfony/mailer ###
      mailer:
        image: schickling/mailcatcher
        ports: [1025, 1080]
    ###< symfony/mailer ###

Toegang tot de webmail
----------------------

.. index::
    single: Symfony CLI;open:local:webmail

Je kunt de webmail openen vanaf een terminal:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:webmail

Of vanuit de web debug toolbar:

.. figure:: screenshots/webmail-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Stuur een reactie in, en je zou een e-mail moeten ontvangen in de webmail interface:

.. figure:: screenshots/webmail.png
    :alt: /
    :align: center
    :figclass: with-browser

Klik op de e-mailtitel in de interface en accepteer of wijs de opmerking af naargelang je behoefte:

.. figure:: screenshots/webmail-rejected.png
    :alt: /
    :align: center
    :figclass: with-browser

Controleer de logs met ``server:log`` wanneer het niet werkt zoals verwacht.

Het beheren van langlopende scripts
-----------------------------------

Het hebben van langlopende scripts gaat gepaard met gedrag waar je je terdege bewust van moet zijn. In tegenstelling tot het PHP-model dat voor HTTP wordt gebruikt, waarbij elk verzoek begint met een schone status, draait de actie van de gebruiker voortdurend op de achtergrond. Elke behandeling van een bericht erft de huidige status, inclusief de cache in het geheugen. Om problemen met Doctrine te voorkomen, worden de entity-managers van Doctrine na de behandeling van een bericht automatisch opgeschoond. Je moet controleren of je eigen services hetzelfde moeten doen of niet.

Asynchroon versturen van e-mailberichten
----------------------------------------

Het kan even duren voordat het e-mailbericht in de message-handler wordt verzonden. Er zou zelfs een exception kunnen optreden bij het versturen. Wanneer er een exception optreedt tijdens de behandeling van een bericht, zal het bericht opnieuw behandeld worden. Maar in plaats van het bericht opnieuw proberen te behandelen, is het beter om in dit geval enkel de e-mail opnieuw te versturen.

We weten al hoe dat moet: zet het e-mailbericht op de bus.

Een ``MailerInterface``-instantie doet het harde werk: wanneer een bus is gedefinieerd, worden e-mailberichten daar op geplaatst in plaats van ze rechtstreeks te versturen. Er zijn hiervoor geen wijzigingen nodig in jouw code.

De bus verstuurd de email reeds asynchroon aangezien dit de standaard Messenger configuratie is:

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 4
    :class: ignore

    framework:
        messenger:
            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async

                # Route your messages to the transports
                App\Message\CommentMessage: async

Zelfs als we hetzelfde transport gebruiken voor reacties en e-mailberichten, hoeft dit nu niet het geval te zijn. Je kunt besluiten om een ander transport te gebruiken om bijvoorbeeld de verschillende prioriteiten van berichten te behandelen. Het gebruik van verschillende transporten geeft je ook de mogelijkheid om verschillende worker-instanties verschillende soorten berichten te laten verwerken. Het is flexibel en ligt in jouw handen.

Testen van e-mails
------------------

Er zijn vele manieren om e-mails te testen.

Je kunt unittests schrijven als je een class per e-mailbericht schrijft (bijvoorbeeld door het extenden van ``Email`` of ``TemplatedEmail``).

De meest voorkomende tests die je zult schrijven zijn echter functionele tests die controleren of sommige acties een e-mail triggeren, en waarschijnlijk tests op de inhoud van de e-mails als ze dynamisch zijn.

Symfony komt met assertions die dergelijke tests vereenvoudigen, hier is een testvoorbeeld dat enkele mogelijkheden laat zien:

.. code-block:: php
    :class: ignore

    public function testMailerAssertions()
    {
        $client = static::createClient();
        $client->request('GET', '/');

        $this->assertEmailCount(1);
        $event = $this->getMailerEvent(0);
        $this->assertEmailIsQueued($event);

        $email = $this->getMailerMessage(0);
        $this->assertEmailHeaderSame($email, 'To', 'fabien@example.com');
        $this->assertEmailTextBodyContains($email, 'Bar');
        $this->assertEmailAttachmentCount($email, 1);
    }

Deze assertions werken wanneer e-mails synchroon of asynchroon worden verzonden.

E-mails versturen op Platform.sh
--------------------------------

.. index::
    single: Platform.sh;Emails
    single: Platform.sh;Mailer
    single: Platform.sh;SMTP
    single: Emails

Er is geen specifieke configuratie benodigd voor Platform.sh. Alle accounts worden geleverd met een Sendgrid-account dat automatisch wordt gebruikt om e-mails te versturen.

.. index::
    single: Symfony CLI;cloud:env:info

.. note::

    Voor de zekerheid worden e-mailberichten standaard enkel op de ``master`` branch verzonden. Schakel SMTP expliciet in als je weet waar je mee bezig bent:

    .. code-block:: terminal

        $ symfony cloud:env:info enable_smtp on

.. sidebar:: Verder gaan

    * `SymfonyCasts Mailer tutorial`_;

    * De `Inky Templating Language documentatie`_;

    * De `omgevingsvariabele-verwerkers`_;

    * De `Symfony Framework Mailer documentatie`_;

    * De `Platform.sh-documentatie over e-mails`_.

.. _`Inky`: https://get.foundation/emails/docs/inky.html
.. _`SymfonyCasts Mailer tutorial`: https://symfonycasts.com/screencast/mailer
.. _`Inky Templating Language documentatie`: https://get.foundation/emails/docs/inky.html
.. _`omgevingsvariabele-verwerkers`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Symfony Framework Mailer documentatie`: https://symfony.com/doc/current/mailer.html
.. _`Platform.sh-documentatie over e-mails`: https://symfony.com/doc/current/cloud/services/emails.html
