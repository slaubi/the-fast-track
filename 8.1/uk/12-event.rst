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

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
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

Додавання цього коду в макет означає, що всі шаблони, що наслідують його, мають визначати змінну ``conferences``, яка має бути створена та передана відповідним контролером.

Оскільки у нас є тільки два контролери, ви *можете* зробити наступне (не застосовуйте зміни до свого коду, оскільки ми дуже скоро дізнаємося про кращий спосіб):

.. code-block:: diff
    :class: ignore

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -21,11 +21,12 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return $this->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,

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

Замість створення конфігураційного файлу, що описує, на які події має реагувати слухач, додайте атрибут ``#[AsEventListener]`` до класу або методу слухача. Це дозволяє слухачам автоматично реєструватися в диспетчері подій Symfony.

Реалізація слухача
-----------------------------------------

.. index::
    single: Event;Listener
    single: Listener
    single: Command;make:listener

Ви вже знаєте цю пісню напам'ять, використовуйте бандл Maker для генерування слухача:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:listener TwigEventListener

Команда запитає вас про те, яку подію ви хочете прослуховувати. Виберіть подію ``Symfony\Component\HttpKernel\Event\ControllerEvent``, яка оголошується безпосередньо перед викликом контролера. Це найкращий час для оголошення глобальної змінної ``conferences``, щоб Twig мав до неї доступ, під час відмальовування шаблону контролером. Оновіть свого слухача наступним чином:

.. code-block:: diff
    :caption: patch_file

    --- i/src/EventListener/TwigEventListener.php
    +++ w/src/EventListener/TwigEventListener.php
    @@ -2,14 +2,22 @@

     namespace App\EventListener;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     final class TwigEventListener
     {
    +    public function __construct(
    +        private Environment $twig,
    +        private ConferenceRepository $conferenceRepository,
    +    ) {
    +    }
    +
         #[AsEventListener]
         public function onControllerEvent(ControllerEvent $event): void
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }
     }

Тепер ви можете додати стільки контролерів, скільки захочете: змінна ``conferences`` завжди буде доступна в Twig.

.. note::

    Ми поговоримо про набагато кращу альтернативу, з точки зору продуктивності, на одному з наступних кроків.

Сортування конференцій за роками та містами
---------------------------------------------------------------------------------

Упорядкування списку конференцій за роками може полегшити перегляд. Ми могли б створити спеціальний метод для отримання і сортування всіх конференцій, але натомість ми перевизначимо реалізацію методу ``findAll()`` за замовчуванням, щоб переконатися, що сортування застосовується скрізь:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/ConferenceRepository.php
    +++ w/src/Repository/ConferenceRepository.php
    @@ -16,6 +16,11 @@ class ConferenceRepository extends ServiceEntityRepository
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
