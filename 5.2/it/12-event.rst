Ascolto degli eventi
====================

Al layout attuale manca una barra di navigazione per tornare alla homepage o per passare da una conferenza all'altra.

Aggiungere una barra di navigazione
-----------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Tutto ciò che va visualizzato su ogni pagina, come una barra di navigazione, dovrebbe far parte del layout di base:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -14,6 +14,15 @@
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

Aggiungere questo codice al layout significa **richiedere** che tutti i template che lo estendono **debbano** definire una variabile ``conferences``, che deve essere creata e passata dai controller.

Avendo solo due controller, si *potrebbe* fare come segue (non applicare la modifica, perché vedremo presto un'alternativa migliore):

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -29,12 +29,13 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return new Response($this->twig->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

Immaginiamo di dover aggiornare decine di controller e fare lo stesso con tutti quelli nuovi. Non è molto pratico. Deve esserci un modo migliore.

Twig ha il concetto di variabili globali. Una *variabile globale* è disponibile in tutti i template. È possibile definirla in un file di configurazione, ma funziona solo per i valori statici. Per aggiungere tutte le conferenze come variabile globale di Twig, creeremo un listener.

Scoprire gli eventi di Symfony
------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony è dotato di un componente EventDispatcher, che si occupa in determinati momenti di *inviare* *eventi*, i quali possono essere ascoltati da *listener*. I listener si agganciano alle parti interne del framework.

Per esempio, alcuni eventi permettono di interagire con il ciclo di vita delle richieste HTTP. Durante la gestione di una richiesta, il dispatcher invia eventi quando una richiesta è stata creata, quando un controller sta per essere eseguito, quando una risposta è pronta per essere inviata o quando è stata sollevata un'eccezione. Un *listener* può ascoltare uno o più eventi ed eseguire una logica basata sull'evento.

Gli eventi sono punti d'estensione ben definiti che rendono il framework più generico ed estensibile. Molti componenti di Symfony come Security, Messenger, Workflow o Mailer li usano ampiamente.

Un altro esempio di eventi e listener in azione è nel ciclo di vita di un comando: è possibile creare un listener per eseguire del codice prima dell'esecuzione di *qualsiasi* comando.

Qualsiasi pacchetto o bundle può anche inviare i propri eventi per rendere il codice estensibile.

Per evitare di avere un file di configurazione che descriva quali eventi un listener vuole ascoltare, creare un *subscriber*. Un subscriber è un listener con un metodo statico ``getSubscribedEvents()`` che restituisce la sua configurazione. Questo permette ai subscriber di registrarsi automaticamente nel dispatcher di Symfony.

Implementare un Subscriber
--------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Ormai conoscete la canzone a memoria, usate MakerBundle per generare un subscriber:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

Il comando chiederà quale evento si vuole ascoltare. Scegliere l'evento ``Symfony\Component\HttpKernel\Event\ControllerEvent``, che viene inviato appena prima che il controller venga chiamato. È il momento migliore per iniettare la variabile globale ``conferences``, in modo che Twig vi abbia accesso quando il controller processerà il template. Aggiorniamo il subscriber come segue:

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
         public function onControllerEvent(ControllerEvent $event)
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }

         public static function getSubscribedEvents()

Ora, è possibile aggiungere tutti i controller che si desidera: la variabile ``conferences`` sarà sempre disponibile nei Twig.

.. note::

    Parleremo di un'alternativa molto migliore dal punto di vista delle prestazioni in uno dei prossimi passi.

Ordinamento delle conferenze per anno e città
----------------------------------------------

Ordinare l'elenco delle conferenze per anno può facilitare la navigazione. Potremmo creare un metodo personalizzato per recuperare e ordinare tutte le conferenze. Optiamo invece per ridefinire l'implementazione predefinita del metodo ``findAll()``,  per assicurarci che l'ordinamento si applichi ovunque:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/ConferenceRepository.php
    +++ b/src/Repository/ConferenceRepository.php
    @@ -19,6 +19,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll()
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         // /**
         //  * @return Conference[] Returns an array of Conference objects
         //  */

Alla fine di questa fase, il sito dovrebbe avere l'aspetto seguente:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Andare oltre

    * Il `flusso di Request-Response <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ nelle applicazioni Symfony;

    * Gli `eventi HTTP predefiniti di Symfony <https://symfony.com/doc/current/reference/events.html>`_;

    * Gli `eventi predefiniti di Symfony Console <https://symfony.com/doc/current/components/console/events.html>`_.
