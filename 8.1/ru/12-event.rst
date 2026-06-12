Обработка событий
=================================

В текущем макете отсутствует шапка с навигационным меню, с помощью которой можно было бы вернуться на главную страницу или перейти на следующую конференцию.

Добавление шапки сайта
------------------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Всё, что необходимо отображать на всех страницах, например, шапка сайта, должно быть частью основного макета:

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

Добавление этого кода в макет означает, что все шаблоны, расширяющие его, должны определить переменную ``conferences``, которую необходимо создать и передать из соответствующих контроллеров.

Поскольку у нас только два контроллера, *можно* поступить следующим образом (это просто пример, не делайте так, скоро мы познакомимся со способом получше):

.. code-block:: diff
    :class: ignore

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -21,12 +21,13 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return $this->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,

Представьте себе, что нам необходимо обновить десятки контроллеров. И с каждым новым контроллером мы будем вынуждены делать то же самое. Это не очень практично. Должен быть способ лучше.

У Twig есть понятие глобальных переменных. *Глобальная переменная* доступна во всех отрисованных шаблонах. Вы можете определить их в конфигурационном файле, но там можно задать только статические значения. Чтобы добавить все конференции в глобальную переменную Twig, мы создадим обработчик событий.

Исследуем события Symfony
-----------------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

В Symfony есть встроенный компонент диспетчера событий (Event Dispatcher). Диспетчер *отправляет* определённые *события* в конкретные моменты времени, а *обработчики* обрабатывают эти события. Обработчики событий позволяют взаимодействовать с внутренностями фреймворка, то есть выполнять некоторый код, в ответ на события, генерируемые в фреймворке.

Например, есть события, которые позволяют взаимодействовать с жизненным циклом HTTP-запросов. При обработке запроса диспетчер генерирует события в следующих случаях: когда создан объект запроса, когда должен быть выполнен контроллер, когда готов HTTP-ответ, либо когда выброшено исключение. *Обработчик* может реагировать на одно или несколько разных событий, и выполнять некоторую логику в зависимости от контекста события.

События — это то, что помогает фреймворку быть более гибким и многофункциональным. Многие компоненты Symfony, такие как Security, Messenger, Workflow или Mailer, активно используют их.

Другим примером встроенных событий и обработчиков в действии является жизненный цикл команды: вы можете создать обработчик, код которого будет выполняться перед запуском *любой* команды.

Любой пакет или бандл может также отправлять свои собственные события, чтобы сделать код расширяемым.

Вместо использования конфигурационного файла, описывающего, на какие события должен реагировать обработчик, добавьте атрибут ``#[AsEventListener]`` на класс или метод обработчика. Это позволяет автоматически регистрировать обработчиков в диспетчере Symfony.

Реализация обработчика
-----------------------------------------

.. index::
    single: Event;Listener
    single: Listener
    single: Command;make:listener

Теперь, когда вы всё знаете, используйте бандл maker, чтобы сгенерировать обработчика:

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:listener TwigEventListener

Команда спросит вас о том, какое событие вы хотите обрабатывать. Выберите событие ``Symfony\Component\HttpKernel\Event\ControllerEvent``, отправляемое непосредственно перед вызовом контроллера. Самое время для определения глобальной переменной ``conferences``, чтобы Twig имел к ней доступ во время отрисовки шаблона контроллером. Обновите вашего обработчика следующим образом:

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

Теперь вы можете добавить сколько угодно контроллеров: переменная ``conferences`` всегда будет доступна в Twig.

.. note::

    Более производительную альтернативу мы рассмотрим позднее.

Сортировка конференций по годам и городам
-----------------------------------------------------------------------------

Представление списка конференций по годам позволит упростить просмотр. Для получения и сортировки всех конференций мы могли бы создать отдельный метод, однако вместо этого переопределим стандартную реализацию метода ``findAll()``, чтобы данная сортировка применялась всегда по умолчанию:

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

В конце этого шага сайт должен выглядеть следующим образом:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Двигаемся дальше

    * `Рабочий процесс запроса-ответа`_ в приложениях Symfony;

    * `Встроенные HTTP-события в Symfony`_;

    * `Встроенные события Symfony Console`_.

.. _`Рабочий процесс запроса-ответа`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`Встроенные HTTP-события в Symfony`: https://symfony.com/doc/current/reference/events.html
.. _`Встроенные события Symfony Console`: https://symfony.com/doc/current/components/console/events.html
