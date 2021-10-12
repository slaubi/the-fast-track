Ascultarea evenimentelor
========================

Șablonului curent îi lipsește un antet de navigare pentru a reveni la pagina principală sau pentru a trece de la o conferință la alta.

Adăugarea unui antet
---------------------

.. index::
    single: Twig;for
    single: Twig;path

Elementele necesare de afișat pe toate paginile web, precum un antet, ar trebui să facă parte din șablonul principal:

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

Adăugarea acestui cod la șablon înseamnă că toate șabloanele care îl extind trebuie să definească o variabilă ``conferences``, care trebuie creată și transmisă din controlerele corespunzătoare.

După cum avem doar două controlere, ai *putea* face următoarele (nu salva modificarea codului, deoarece vom învăța o modalitate mai bună foarte curând):

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

Imaginează-ți că trebuie să actualizezi zeci de controlere. Și va trebui să faci la fel pentru toate cele noi. Acest lucru nu este tocmai practic. Trebuie să existe o metodă mai bună.

Twig are noțiunea de variabile globale. O *variabilă globală* este disponibilă în toate șabloanele redate. Poți să le definești într-un fișier de configurare, dar funcționează numai pentru valori statice. Pentru a adăuga toate conferințele ca o variabilă globală Twig, vom crea o clasă de tip Listener.

Descoperirea evenimentelor Symfony
----------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony conține o componentă numită Event Dispatcher. Un dispatcher *expediază (dispatch)* anumite *evenimente (events)* în momente specifice pe care *clasele Listener* le pot asculta. Clasele Listener sunt cârlige din cadrul componentelor interne ale framework-ului.

De exemplu, unele evenimente îți permit să interacționezi cu ciclul de viață al solicitărilor HTTP. În timpul procesării unei solicitări, dispecerul expediază evenimente când a fost creată o solicitare, când un controler este pe punctul de a fi executat, când un răspuns este gata de trimis către browser sau când a fost aruncată o excepție. O clasă *Listener* poate asculta unul sau mai multe evenimente și poate executa unele acțiuni logice bazate pe contextul evenimentului.

Evenimentele sunt puncte de extensie bine definite care fac framework-ul mai generic și extensibil. Multe componente Symfony precum Security, Messenger, Workflow sau Mailer le folosesc pe scară largă.

Un alt exemplu de clase Listener și evenimente încorporate în Symfony este ciclul de viață al unei comenzi: poți crea o clasă Listener pentru a executa codul tău înainte ca *orice* comandă să fie executată.

Orice pachet bundle poate avea propriile evenimente pentru a face codul extensibil.

Pentru a evita crearea unui fișier de configurare care descrie explicit evenimentele pe care o clasă Listener le va asculta, poți crea o clasă de tip *Subscriber*. Un *Subscriber* este un *Listener* cu o metodă statică ``getSubscribeEvents()`` care returnează configurația sa. Aceasta permite abonaților să fie înregistrați automat în dispecerul Symfony.

Implementarea unui Subscriber
-----------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Cunoști deja pașii, utilizează MakerBundle pentru a genera un Subscriber:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

Comanda te va întreaba ce eveniment dorești să asculți. Alege evenimentul ``Symfony\Component\HttpKernel\Event\ControllerEvent``, care este expediat chiar înainte de apelarea controlerului. Este cel mai bun moment pentru a injecta variabila globală ``conferences``, astfel încât Twig va avea acces la ea atunci când controlerul va reda șablonul. Actualizeză *Subscriber-ul* după cum urmează:

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

Acum poți adăuga câte controlere dorești: variabila ``conferences`` va fi întotdeauna disponibilă în Twig.

.. note::

    Vom vorbi despre o alternativă mult mai bună din punct de vedere al performanței într-un pas ulterior.

Sortarea conferințelor în funcție de an și oraș
----------------------------------------------------

Ordonarea conferințelor în funcție de an poate facilita navigarea. Am putea crea o metodă personalizată pentru a prelua și sorta toate conferințele, dar în schimb, vom trece la suprascrierea metodei ``findAll()`` pentru a fi siguri că sortarea se aplică peste tot:

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

La sfârșitul acestui pas, site-ul web ar trebui să arate astfel:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Mergând mai departe

    * `Fluxul de solicitare-răspuns (Request-Response) <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ în aplicațiile Symfony;

    * `Evenimentele HTTP integrate în Symfony <https://symfony.com/doc/current/reference/events.html>`_;

    * `Evenimentele încorporate în Symfony Console <https://symfony.com/doc/current/components/console/events.html>`_.
