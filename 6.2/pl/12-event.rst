Nasłuchiwanie zdarzeń
=======================

W obecnym szablonie brakuje nagłówka nawigacyjnego, który umożliwi powrót na stronę główną lub przejście do kolejnej konferencji.

Dodawanie nagłówka strony
---------------------------

.. index::
    single: Twig;for
    single: Twig;path

Elementy wyświetlane na wszystkich stronach aplikacji, jak nagłówek strony, powinny zostać zawarte w szablonie bazowym:

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

Dodanie tego kodu do szablonu oznacza, że wszystkie szablony rozszerzające go muszą zdefiniować zmienną o nazwie ``conferences``, której wartość musi zostać utworzona i przekazana z poziomu kontrolerów.

Ponieważ mamy tylko dwa kontrolery, *możesz* wykonać następujące czynności (nie stosuj zmiany w kodzie, ponieważ wkrótce nauczymy się lepszego sposobu):

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
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

Wyobraź sobie, że musisz wprowadzić zmianę w dziesiątkach istniejących kontrolerów, a także we wszystkich nowo utworzonych. Nie jest to zbyt praktyczne, musi istnieć lepszy sposób.

Twig pozwala na tworzenie zmiennych globalnych. *Zmienna globalna* jest dostępna we wszystkich renderowanych szablonach. Można je zdefiniować w pliku konfiguracyjnym, ale zadziała to tylko w przypadku wartości statycznych. Aby przekazać wszystkie konferencje jako zmienną globalną w Twig, stworzymy nasłuchiwacz zdarzeń (ang. event listener).

Odkrywanie zdarzeń Symfony
---------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony posiada wbudowany komponent dyspozytora zdarzeń (ang. event dispatcher). Dyspozytor zdarzeń w odpowiednich momentach emituje określone *zdarzenia*, które mogą być nasłuchiwane przez nasłuchiwaczy zdarzeń. Nasłuchiwacze zdarzeń umożliwiają w tym przypadku wywołanie kodu w odpowiedzi na zdarzenia wyemitowane przez wewnętrzne mechanizmy frameworka.

Na przykład, niektóre zdarzenia pozwalają na interakcję z cyklem życia żądań HTTP. Podczas obsługi żądania dyspozytor zdarzeń emituje zdarzenia, gdy żądanie zostało utworzone, gdy kod kontrolera ma zostać wykonany, gdy odpowiedź jest gotowa do wysłania, lub gdy został rzucony wyjątek. *Nasłuchiwacz zdarzeń* może oczekiwać jednego lub kilku zdarzeń i wykonywać działania w oparciu o kontekst przekazany w obiekcie zdarzenia.

Zdarzenia są elementami frameworka, które czynią go łatwo rozszerzalnym i bardziej elastycznym. Wiele komponentów Symfony, takich jak Security, Messenger, Workflow czy Mailer, używa ich w szerokim zakresie.

Inny przykład użycia wbudowanego we framework mechanizmu zdarzeń i nasłuchiwaczy zdarzeń powiązany jest z cyklem życia poleceń (ang. commands): możesz utworzyć nasłuchiwacz zdarzeń, który wykona kod przed uruchomieniem *jakiegolwiek* polecenia Symfony.

Dowolny rodzaj rozszerzenia, jak paczka lub pakiet, może emitować własne typy zdarzeń, co czyni jego kod łatwo rozszerzalnym.

Aby uniknąć posiadania pliku konfiguracyjnego opisującego zdarzenia, których nasłuchiwacz zdarzeń chce nasłuchiwać, utwórz *subskrybenta zdarzeń* (ang. event subscriber). Subskrybent zdarzeń jest rodzajem nasłuchiwacza zdarzeń posiadającym statyczną metodę ``getSubscribedEvents()``, która zwraca jego konfigurację. Dzięki temu subskrybent może być automatycznie zarejestrowany w dyspozytorze zdarzeń Symfony.

Implementacja subskrybenta zdarzeń
-----------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Znasz już tę śpiewkę – użyj Maker Bundle, aby wygenerować subskrybenta zdarzeń:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

Polecenie zapyta o to, jakich zdarzeń chcesz nasłuchiwać. Wybierz zdarzenie typu ``Symfony\Component\HttpKernel\Event\ControllerEvent``, które jest emitowane tuż przed wykonaniem kodu kontrolera. Jest to najlepszy moment na wstrzyknięcie globalnej zmiennej ``conferences`` w taki sposób, aby Twig miał do niej dostęp, kiedy kontroler będzie renderował szablon. Zaktualizuj swojego subskrybenta zdarzeń w następujący sposób:

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

Teraz możesz dodać dowolną liczbę kontrolerów: zmienna ``conferences`` będzie zawsze dostępna w szablonach Twig.

.. note::

    W dalszej części książki rozważymy znacznie lepsze i bardziej wydajne rozwiązanie.

Sortowanie konferencji według roku i miasta
--------------------------------------------

Sortowanie listy konferencji według roku ułatwi jej przeglądanie. Moglibyśmy stworzyć własną metodę pobierania i sortowania wszystkich konferencji, ale zamiast tego zastąpimy domyślną implementację metody ``findAll()``, aby upewnić się, że sortowanie będzie użyte we wszystkich wymaganych miejscach:

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
         public function save(Conference $entity, bool $flush = false): void
         {
             $this->getEntityManager()->persist($entity);

Na koniec tego etapu strona aplikacji powinna wyglądać następująco:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Idąc dalej

    * `Cykl życia Request-Response`_ w aplikacjach Symfony;

    * `Wbudowane zdarzenia Symfony HTTP`_;

    * `Wbudowane zdarzenia Symfony Console`_.

.. _`Cykl życia Request-Response`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`Wbudowane zdarzenia Symfony HTTP`: https://symfony.com/doc/current/reference/events.html
.. _`Wbudowane zdarzenia Symfony Console`: https://symfony.com/doc/current/components/console/events.html
