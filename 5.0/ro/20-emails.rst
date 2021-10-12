Expedierea e-mail-urilor administratorilor
==========================================

.. index::
    single: Components;Mailer
    single: Mailer
    single: Emails

Pentru a asigura feedback de înaltă calitate, administratorul trebuie să modereze toate comentariile. Când un comentariu este în starea ``ham`` sau ``potential_spam``, un *e-mail* ar trebui expediat către administrator cu două link-uri: unul pentru a accepta comentariul și unul pentru a-l respinge.

Mai întâi, instalează componenta Symfony Mailer:

.. code-block:: bash

    $ symfony composer req mailer

Setarea unui e-mail pentru administrator
----------------------------------------

Pentru a salva emailul administratorului, utilizează un parametru în container. În scop demonstrativ, îi permitem să fie setat și prin intermediul unei variabile de mediu (nu ar trebui să fie nevoie în „viața reală”). Pentru a ușura injectarea în serviciile care au nevoie de emailul administratorului, definește o setare ``bind`` în configurarea containerului:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -4,6 +4,7 @@
     # Put parameters here that don't need to change on each machine where the app is deployed
     # https://symfony.com/doc/current/best_practices/configuration.html#application-related-configuration
     parameters:
    +    default_admin_email: admin@example.com

     services:
         # default configuration for services in *this* file
    @@ -13,6 +14,7 @@ services:
             bind:
                 $photoDir: "%kernel.project_dir%/public/uploads/photos"
                 $akismetKey: "%env(AKISMET_KEY)%"
    +            $adminEmail: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

O variabilă de mediu ar putea fi „procesată” înainte de a fi utilizată. Aici, folosim procesorul ``default`` pentru a reveni la valoarea parametrului ``default_admin_email`` dacă variabila de mediu ``ADMIN_EMAIL`` nu există.

Expedierea unui e-mail de notificare
------------------------------------

Pentru a expedia un e-mail, poți alege între mai multe abstractizări ale clasei ``Email``; de la `` Message``, cel mai mic nivel, la ``NotificationEmail``, cel mai înalt. Probabil vei folosi cel mai mult clasa ``Email``, dar `` NotificationEmail`` este alegerea perfectă pentru e-mailurile interne.

În manipulatorul de mesaje, să înlocuim logica de validare automată:

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

``MailerInterface`` este punctul principal de intrare și permite expedierea emailurilor prin intermediul funcției ``send()``.

Pentru a expedia un e-mail, avem nevoie de un expeditor (antetul ``From``/``Sender``). În loc să setăm opțiunea explicit pe instanța Email o definim la nivel global:

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

Extinderea șablonului de e-mail de notificare
----------------------------------------------

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;url

Modelul de e-mail de notificare este moștenit de la șablonul de e-mail de notificare implicit care vine cu Symfony:

.. code-block:: twig
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

Șablonul înlocuiește câteva blocuri pentru a personaliza mesajul e-mailului și pentru a adăuga câteva link-uri care permit administratorului să accepte sau să respingă un comentariu. Orice argument de rută care nu este un parametru de rută valid este adăugat ca un element de șir de interogare (URL-ul de respingere pare a fi ``/admin/comment /review/42?reject=true``).

Șablonul implicit ``NotificationEmail`` utilizează `Inky <https://get.foundation/emails/docs/inky.html>`_ în loc de HTML pentru a proiecta e-mailuri. Acesta ajută la crearea de e-mailuri responsive, compatibile cu toți clienții de e-mail populari.

Pentru o compatibilitate maximă cu cititorii de e-mail, aspectul bazei de notificare conține în mod implicit toate foile de stil (prin pachetul CSS inliner).

Aceste două caracteristici fac parte din extensiile opționale Twig care trebuiesc instalate:

.. code-block:: bash

    $ symfony composer req "twig/cssinliner-extra:^3" "twig/inky-extra:^3"

Generarea URL-urilor absolute într-o comandă
----------------------------------------------

.. index::
    single: Twig;Link
    single: Link

În e-mailuri, generează adrese URL cu ``url()`` în loc de ``path()``, deoarece ai nevoie de cele absolute (cu schema și gazda).

E-mailul este trimis de la instrumentul de gestionare a mesajelor, într-un context de consolă. Generarea URL-urilor absolute într-un context Web este mai ușoară, deoarece știm schema și domeniul paginii curente. Nu este cazul într-un context de consolă.

Definește numele de domeniu și schema de utilizat în mod explicit:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -5,6 +5,11 @@
     # https://symfony.com/doc/current/best_practices/configuration.html#application-related-configuration
     parameters:
         default_admin_email: admin@example.com
    +    default_domain: '127.0.0.1'
    +    default_scheme: 'http'
    +
    +    router.request_context.host: '%env(default:default_domain:SYMFONY_DEFAULT_ROUTE_HOST)%'
    +    router.request_context.scheme: '%env(default:default_scheme:SYMFONY_DEFAULT_ROUTE_SCHEME)%'

     services:
         # default configuration for services in *this* file

Variabilele de mediu ``SYMFONY_DEFAULT_ROUTE_HOST`` și ``SYMFONY_DEFAULT_ROUTE_PORT`` sunt setate automat atunci când se utilizează CLI ``symfony`` și se determină în baza configurației de pe SymfonyCloud.

Setarea căilor către un controler
-----------------------------------

Ruta ``review_comment`` nu există încă, hai să creăm un controler de administrare care să o gestioneze:

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

        /**
         * @Route("/admin/comment/review/{id}", name="review_comment")
         */
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

            return $this->render('admin/review.html.twig', [
                'transition' => $transition,
                'comment' => $comment,
            ]);
        }
    }

URL-ul comentariului de recenzie începe cu ``/admin/`` pentru a-l proteja cu firewall-ul definit într-un pas anterior. Administratorul trebuie să fie autentificat pentru a accesa această resursă.

În loc să creăm o instanță ``Response``, am folosit ``render()``, o metodă de comandă furnizată de clasa de bază a controlerului ``AbstractController``.

.. index::
    single: Twig;extends
    single: Twig;block

Când revizuirea este terminată, un șablon scurt mulțumește administratorului pentru munca grea:

.. code-block:: twig
    :caption: templates/admin/review.html.twig

    {% extends 'base.html.twig' %}

    {% block body %}
        <h2>Comment reviewed, thank you!</h2>

        <p>Applied transition: <strong>{{ transition }}</strong></p>
        <p>New state: <strong>{{ comment.state }}</strong></p>
    {% endblock %}

Utilizarea unui Mail Catcher
----------------------------

.. index::
    single: Docker;Mail Catcher

În loc să folosești un server SMTP „real” sau un furnizor terț pentru a expedia e-mailuri, să folosim un captator de e-mail. Un captator de e-mail furnizează un server SMTP care nu livrează e-mailurile, dar le pune la dispoziție printr-o interfață Web:

.. code-block:: diff

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -16,3 +16,7 @@ services:
         rabbitmq:
             image: rabbitmq:3.7-management
             ports: [5672, 15672]
    +
    +    mailer:
    +        image: schickling/mailcatcher
    +        ports: [1025, 1080]

Reporniți containerele pentru a adăuga captatorul de e-mail:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

Accesarea site-ului Webmail
---------------------------

.. index::
    single: Symfony CLI;open:local:webmail

Poți deschide e-mailul de la un terminal:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:webmail

Sau din bara de instrumente de depanare web:

.. figure:: screenshots/webmail-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Expediază un comentariu, ar trebui să primești un e-mail în interfața de e-mail:

.. figure:: screenshots/webmail.png
    :alt: /
    :align: center
    :figclass: with-browser

Execută un clic pe titlul de e-mail de pe interfață și acceptă sau respinge comentariul după cum consideri potrivit:

.. figure:: screenshots/webmail-rejected.png
    :alt: /
    :align: center
    :figclass: with-browser

Verifică jurnalele cu ``server:log`` dacă acest lucru nu funcționează așa cum te aștepți.

Gestionarea scripturilor pe termen lung
---------------------------------------

Dacă ai scripturi de lungă durată, există comportamente de care ar trebui să fii conștient. Spre deosebire de modelul PHP utilizat pentru HTTP unde fiecare cerere începe cu o stare curată, consumatorul de mesaje rulează continuu în fundal. Fiecare manipulare a unui mesaj moștenește starea curentă, inclusiv memoria cache. Pentru a evita orice problemă cu Doctrine, administratorii entității sale sunt șterse automat după manipularea unui mesaj. Ar trebui să verifici dacă propriile servicii trebuie să facă același lucru sau nu.

Expedierea e-mailurilor în mod asincron
----------------------------------------

E-mailul expediat în gestionarea mesajelor ar putea dura ceva timp pentru a fi expediat. S-ar putea chiar să arunce o excepție. În cazul în care o excepție este aruncată în timpul manipulării unui mesaj, acesta va fi reîncercat. Dar, în loc să încerci din nou să consumi mesajul de comentariu, ar fi mai bine să încerci să expediezi e-mailul.

Știm deja cum să facem asta: expediază mesajul de e-mail în bus.

O instanță ``MailerInterface`` execută partea cea mai dificilă: când un bus este definit, acesta trimite mesaje de email pe el în loc să le expedieze. Nu este nevoie de modificări în cod.

Însă, acum, bus-ul expediază emailul în mod sincron, deoarece nu am configurat coada de mesaje pe care vrem să o utilizăm pentru emailuri. Să folosim din nou RabbitMQ:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -19,3 +19,4 @@ framework:
             routing:
                 # Route your messages to the transports
                 App\Message\CommentMessage: async
    +            Symfony\Component\Mailer\Messenger\SendEmailMessage: async

Chiar dacă folosim același transport (RabbitMQ) pentru mesaje de comentarii și mesaje de e-mail, nu trebuie să fie cazul. Poți decide să utilizezi un alt transport pentru a gestiona diferite priorități de mesaj, de exemplu. Utilizarea diferitelor transporturi îți oferă, de asemenea, posibilitatea de a avea diferite utilaje care lucrează diferite tipuri de mesaje. Este flexibil și depinde de tine.

Testarea e-mailurilor
---------------------

Există multe modalități de testare a e-mailurilor.

Poți scrie teste unitare dacă scrii o clasă per e-mail (extinzând ``Email`` sau ``TemplateEmail``, de exemplu).

Cele mai comune teste pe care le vei scrie sunt teste funcționale care verifică dacă unele acțiuni declanșează un e-mail și, probabil, teste despre conținutul e-mailurilor, dacă acestea sunt dinamice.

Symfony vine cu afirmații care ușurează astfel de teste:

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

Aceste afirmații funcționează atunci când e-mailurile sunt expediate sincron sau asincron.

Expedierea e-mail-urilor pe SymfonyCloud
----------------------------------------

.. index::
    single: SymfonyCloud;Emails
    single: SymfonyCloud;Mailer
    single: SymfonyCloud;SMTP
    single: Emails

Nu există o configurație specifică pentru SymfonyCloud. Toate conturile vin cu un cont Sendgrid care este automat utilizat pentru a expedia e-mailuri.

Mai trebuie să actualizezi configurația SymfonyCloud pentru a include extensia PHP ``xsl`` de care are nevoie Inky:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - xsl
             - amqp
             - redis
             - pdo_pgsql

.. index::
    single: Symfony CLI;env:setting:set

.. note::

    Pentru a fi în siguranță, e-mailurile *sunt* expediate în mod implicit doar pe ramura ``master``. Activează în mod explicit SMTP pe ramurile non-``master`` dacă știi ce faci:

    .. code-block:: bash

        $ symfony env:setting:set email on

.. sidebar:: Mergând mai departe

    * `Tutorial SymfonyCasts Mailer <https://symfonycasts.com/screencast/mailer>`_;

    * Documentația `limbajului de șabloane Inky <https://get.foundation/emails/docs/inky.html>`_;

    * `Procesoare variabile de mediu <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * Documentația `Symfony Framework Mailer <https://symfony.com/doc/current/mailer.html>`_;

    * The `SymfonyCloud documentation about Emails <https://symfony.com/doc/current/cloud/services/emails.html>`_.
