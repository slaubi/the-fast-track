Mit Events arbeiten
===================

Dem aktuellen Layout fehlt eine Navigation, um zur Homepage zurückzukehren oder von einer Konferenz zur nächsten zu wechseln.

Einen Website-Header hinzufügen
--------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Alles, was auf allen Webseiten angezeigt werden soll, wie z. B. ein Header, sollte Teil des Haupt-Basislayouts sein:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -12,6 +12,15 @@
             {% endblock %}
         </head>
         <body>
    +        <header>
    +            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    +            <ul>
    +            {% for conference in conferences %}
    +                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +            {% endfor %}
    +            </ul>
    +            <hr />
    +        </header>
             {% block body %}{% endblock %}
         </body>
     </html>

Das Hinzufügen dieses Codes zum Layout bedeutet, dass alle Templates, die es erweitern, eine ``conferences``-Variable definieren müssen, die von ihren Controllern erstellt und übergeben werden muss.

Da wir nur zwei Controller haben, "könntest" Du Folgendes tun (ändere das jetzt nicht in deinem Code - wir werden schon bald eine bessere Art und Weise lernen):

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -21,12 +21,13 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return $this->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,

Stelle Dir vor, Du müsstest Dutzende von Controllern aktualisieren. Und das Gleiche bei allen neuen Controllern tun. Das ist nicht besonders praktisch. Es muss einen besseren Weg geben.

Twig bietet die Möglichkeit globale Variablen zu definieren. Eine *globale Variable* ist in allen gerenderten Vorlagen verfügbar. Du kannst sie in einer Konfigurationsdatei definieren, aber das funktioniert nur bei statischen Werten. Um alle Konferenzen als globale Variable zu Twig hinzuzufügen, werden wir einen Listener erstellen.

Symfony Events entdecken
------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony ist mit einer Event Dispatcher Komponente ausgestattet. Ein Dispatcher *verteilt* bestimmte *Events* zu bestimmten Zeiten, die ein *Listener* abonnieren kann. Listener sind Hooks im Inneren des Frameworks.

Einige Events erlauben es Dir beispielsweise, mit dem Lifecycle von HTTP-Requests zu interagieren. Während der Bearbeitung eines Requests sendet der Dispatcher Events, sobald ein Request erstellt wurde, ein Controller aufgerufen werden soll, eine Response zum Senden bereit ist oder eine Exception geworfen wurde. Ein *Listener* kann auf ein oder mehrere Events reagieren und Logik basierend auf dem Eventkontext ausführen.

Events sind klar definierte Erweiterungspunkte, die das Framework generischer und erweiterbarer machen. Viele Symfony-Komponenten wie Security, Messenger, Workflow oder Mailer verwenden sie häufig.

Ein weiteres eingebautes Beispiel für Events und Listener ist der Lifecycle eines Befehls: Du kannst einen Listener erstellen, um Code vor *jedem* Befehl auszuführen.

Jedes Paket oder Bundle kann auch eigene Events auslösen, um seinen Code erweiterbar zu machen.

Damit du nicht alle Events und Listener in einer Konfigurationsdatei beschreiben musst, kannst du einen *Subscriber* erstellen. Ein Subscriber ist ein Listener mit einer statischen ``getSubscribedEvents()``-Methode, die seine Konfiguration zurückgibt. Dadurch können Subscriber automatisch im Symfony Dispatcher registriert werden und Events abonnieren.

Einen Subscriber implementieren
-------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Du kennst das Lied bestimmt schon auswendig, verwende das Maker-Bundle, um einen Subscriber zu generieren:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

Der Befehl fragt Dich, über welches Event Du informiert werden möchtest. Wähle das ``Symfony\Component\HttpKernel\Event\ControllerEvent``-Event, welches kurz vor dem Aufruf eines Controllers ausgelöst wird. Dies ist der beste Zeitpunkt, die globale ``conferences``-Variable einzuspeisen, damit Twig Zugriff darauf hat, wenn der Controller das Template rendert. Passe Deinen Subscriber wie folgt an:

.. code-block:: diff
    :caption: patch_file

    --- a/src/EventSubscriber/TwigEventSubscriber.php
    +++ b/src/EventSubscriber/TwigEventSubscriber.php
    @@ -2,14 +2,25 @@

     namespace App\EventSubscriber;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\EventSubscriberInterface;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     class TwigEventSubscriber implements EventSubscriberInterface
     {
    +    private $twig;
    +    private $conferenceRepository;
    +
    +    public function __construct(Environment $twig, ConferenceRepository $conferenceRepository)
    +    {
    +        $this->twig = $twig;
    +        $this->conferenceRepository = $conferenceRepository;
    +    }
    +
         public function onControllerEvent(ControllerEvent $event): void
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }

         public static function getSubscribedEvents(): array

Jetzt kannst Du beliebig viele Controller hinzufügen: Die ``conferences``-Variable wird in Twig immer verfügbar sein.

.. note::

    Wir werden in einem späteren Schritt über eine viel bessere Alternative in Bezug auf Performance sprechen.

Konferenzen nach Jahr und Stadt sortieren
-----------------------------------------

Eine nach Jahren sortierte Konferenzliste kann das Durchsuchen erleichtern. Wir könnten eine spezifische Methode erstellen, um alle Konferenzen abzurufen und zu sortieren, stattdessen werden wir jedoch die Standardimplementierung der ``findAll()``-Methode überschreiben. Auf diese Weise stellen wir sicher, dass die Sortierung überall angewendet wird:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/ConferenceRepository.php
    +++ b/src/Repository/ConferenceRepository.php
    @@ -21,6 +21,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll(): array
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         //    /**
         //     * @return Conference[] Returns an array of Conference objects
         //     */

Nach diesem Schritt sollte die Seite wie folgt aussehen:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Weiterführendes

    * Der `Request-Response Ablauf`_ in Symfony-Anwendungen;

    * Die `eingebauten Symfony HTTP-Events`_;

    * Die `eingebauten Events der Symfony Console`_.

.. _`Request-Response Ablauf`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`eingebauten Symfony HTTP-Events`: https://symfony.com/doc/current/reference/events.html
.. _`eingebauten Events der Symfony Console`: https://symfony.com/doc/current/components/console/events.html
