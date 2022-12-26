Прослуховування подій
=========================================

У поточному макеті відсутня шапка з навігаційним меню, що дозволяє повернутися на головну сторінку або перейти з однієї конференції на іншу.

Додавання шапки веб-сайту
-----------------------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Все, що має бути відображено на всіх веб-сторінках, наприклад шапка сайту, має бути частиною основного базового макета:

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

Додавання цього коду в макет означає, що всі шаблони, що наслідують його, мають визначати змінну ``conferences``, яка має бути створена та передана відповідним контролером.

Оскільки у нас є тільки два контролери, ви *можете* зробити наступне (не застосовуйте зміни до свого коду, оскільки ми дуже скоро дізнаємося про кращий спосіб):

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

Уявіть, що вам потрібно оновити десятки контролерів. І робити те ж саме для всіх нових. Це не дуже практично. Має бути кращий спосіб.

У Twig є поняття глобальних змінних. *Глобальна змінна* доступна в усіх відмальованих шаблонах. Ви можете визначити їх у файлі конфігурації, але це працює лише для статичних значень. Щоб додати всі конференції як глобальну змінну Twig, ми створимо слухача.

Дослідження подій у Symfony
--------------------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony поставляється з вбудованим компонентом Event Dispatcher. Диспетчер *оголошує* про певні *події* у визначений час, які можуть слухати *слухачі*. Слухачі — це гачки (англ. hooks), що дозволяють взаємодіяти з внутрішніми механізмами фреймворку.

Наприклад, є деякі події що дозволяють взаємодіяти з життєвим циклом HTTP-запитів. Під час обробки запиту диспетчер оголошує події у наступних випадках: коли запит створено, коли планується виконати контролер, коли відповідь готова до відправлення або коли було кинуто виняток. *Слухач* може слухати одну або декілька подій і виконувати певну логіку, виходячи з контексту події.

Події — це чітко визначені точки розширення, які роблять фреймворк більш функціональним та гнучким. Багато компонентів Symfony, такі як Security, Messenger, Workflow або Mailer, широко використовують їх.

Іншим вбудованим прикладом подій і слухачів у дії є життєвий цикл команди: ви можете створити слухача для виконання коду перед виконанням *будь-якої* команди.

Будь-який пакет або бандл також може оголошувати свої власні події, щоб зробити код більш розширюваним.

Замість створення конфігураційного файлу, що описує, на які події має реагувати слухач, створіть *підписника*. Підписник — це слухач зі статичним методом ``getSubscribedEvents()``, що повертає власну конфігурацію. Це дозволяє підписникам автоматично реєструватися в диспетчері подій Symfony.

Реалізація підписника
-----------------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Ви вже знаєте цю пісню напам'ять, використовуйте бандл Maker для генерування підписника:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

Команда запитає вас про те, яку подію ви хочете прослуховувати. Виберіть подію ``Symfony\Component\HttpKernel\Event\ControllerEvent``, яка оголошується безпосередньо перед викликом контролера. Це найкращий час для оголошення глобальної змінної ``conferences``, щоб Twig мав до неї доступ, під час відмальовування шаблону контролером. Оновіть свого підписника наступним чином:

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

Тепер ви можете додати стільки контролерів, скільки захочете: змінна ``conferences`` завжди буде доступна в Twig.

.. note::

    Ми поговоримо про набагато кращу альтернативу, з точки зору продуктивності, на одному з наступних кроків.

Сортування конференцій за роками та містами
---------------------------------------------------------------------------------

Упорядкування списку конференцій за роками може полегшити перегляд. Ми могли б створити спеціальний метод для отримання і сортування всіх конференцій, але натомість ми перевизначимо реалізацію методу ``findAll()`` за замовчуванням, щоб переконатися, що сортування застосовується скрізь:

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

В кінці цього кроку веб-сайт повинен виглядати наступним чином:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Йдемо далі

    * `Життєвий цикл Request-Response`_ у застосунках Symfony.

    * `Вбудовані HTTP-події Symfony`_;

    * `Вбудовані події Symfony Console`_.

.. _`Життєвий цикл Request-Response`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`Вбудовані HTTP-події Symfony`: https://symfony.com/doc/current/reference/events.html
.. _`Вбудовані події Symfony Console`: https://symfony.com/doc/current/components/console/events.html
