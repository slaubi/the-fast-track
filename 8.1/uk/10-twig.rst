Створення інтерфейсу користувача
==============================================================

.. index::
    single: Twig
    single: Templates

Тепер все готово для створення першої версії інтерфейсу користувача веб-сайту. Ми не будемо робити його красивим. Для початку, зробимо його функціональним.

Пам’ятаєте як нам довелося екранувати рядок у контролері, щоб уникнути проблем з безпекою, коли ми робили пасхалку? Ми не будемо використовувати PHP для наших шаблонів з цієї ж причини. Натомість будемо використовувати Twig. Окрім безпечної обробки даних, `Twig`_ надає ще багато корисних можливостей, які ми будемо використовувати (наприклад, наслідування шаблонів).

Використання Twig для шаблонів
-----------------------------------------------------

.. index::
    single: Twig;Layout
    single: Twig;block

Усі сторінки на веб-сайті матимуть однаковий *макет*. Під час встановлення Twig автоматично створюється каталог ``templates/``, а також зразок макета ``base.html.twig``.

.. code-block:: html+twig
    :caption: templates/base.html.twig
    :class: ignore

    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
            {% block stylesheets %}
            {% endblock %}

            {% block javascripts %}
                {% block importmap %}{{ importmap('app') }}{% endblock %}
            {% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
        </body>
    </html>

У макеті можуть бути визначені спеціальні елементи  ``block``, які є місцями, де *дочірні шаблони*, які *наслідують* макет, додають власний вміст.

.. index::
    single: Twig;extends
    single: Twig;for

Створімо шаблон для головної сторінки проекту у файлі ``templates/conference/index.html.twig``:

.. code-block:: html+twig
    :caption: templates/conference/index.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook{% endblock %}

    {% block body %}
        <h2>Give your feedback!</h2>

        {% for conference in conferences %}
            <h4>{{ conference }}</h4>
        {% endfor %}
    {% endblock %}

Шаблон *наслідує* ``base.html.twig`` і перевизначає блоки ``title`` та ``body``.

.. index::
    single: Twig;Syntax

Позначення ``{% %}`` у шаблоні вказує на *дії * та *структуру*.

Позначення ``{{ }}`` використовується для *відображення* чого-небудь. ``{{ conference }}`` відображає рядкове представлення конференції (результат виклику ``__toString`` для об'єкта ``Conference``).

Використання Twig у контролері
-----------------------------------------------------

Оновіть контролер, щоб відмалювати шаблон Twig:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,22 +2,19 @@

     namespace App\Controller;

    +use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    +use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response(<<<EOF
    -            <html>
    -                <body>
    -                    <img src="/images/under-construction.gif" />
    -                </body>
    -            </html>
    -            EOF
    -        );
    +        return new Response($twig->render('conference/index.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
         }
     }

Тут багато чого відбувається.

Щоб відмалювати шаблон, нам потрібен об’єкт Twig — ``Environment`` (головна точка входу Twig). Зверніть увагу, щоб отримати екземпляр Twig, достатньо вказати його тип в аргументах методу контролера. Symfony достатньо розумний і знає як автоматично впровадити залежність потрібного типу.

Нам також потрібен репозиторій конференцій, щоб отримати всі конференції з бази даних.

У коді контролера метод ``render()`` відмальовує шаблон і передає йому масив змінних. Ми передаємо список об'єктів ``Conference`` у  якості змінної ``conferences``.

Контролер — це звичайний клас PHP. Нам навіть не потрібно наслідувати клас ``AbstractController``, якщо ми хочемо мати найбільш явний контроль залежностей. Ви можете видалити його (але не робіть цього, оскільки ми будемо використовувати деякі його методи, на наступних кроках).

Створення сторінки для конференції
-----------------------------------------------------------------

Кожна конференція має мати власну сторінку з коментарями. Додавання нової сторінки — це додавання контролера, визначення маршруту для нього і створення відповідного шаблону.

Додайте метод ``show()`` у ``src/Controller/ConferenceController.php``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,6 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Conference;
    +use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    @@ -17,4 +20,13 @@ final class ConferenceController extends AbstractController
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    +
    +    #[Route('/conference/{id}', name: 'conference')]
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    {
    +        return new Response($twig->render('conference/show.html.twig', [
    +            'conference' => $conference,
    +            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +        ]));
    +    }
     }

Цей метод має особливу поведінку, яку ми ще не бачили. Ми вказуємо, що екземпляр ``Conference`` має бути впроваджений у метод. Проте, в базі даних може бути багато об'єктів. Атрибут ``#[MapEntity]`` вказує Symfony отримати потрібний об'єкт, ґрунтуючись на ``{id}``, переданому в рядку запиту (``id`` є первинним ключем таблиці ``conference`` в базі даних).

Отримати коментарі, що стосуються конференції, можна за допомогою методу ``findBy()``, який приймає критерії у якості першого аргументу.

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;for
    single: Twig;if
    single: Twig;else
    single: Twig;asset
    single: Twig;format_datetime
    single: Twig;length

Останнім кроком є створення файлу ``templates/conference/show.html.twig``:

.. code-block:: html+twig
    :caption: templates/conference/show.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

    {% block body %}
        <h2>{{ conference }} Conference</h2>

        {% if comments|length > 0 %}
            {% for comment in comments %}
                {% if comment.photofilename %}
                    <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
                {% endif %}

                <h4>{{ comment.author }}</h4>
                <small>
                    {{ comment.createdAt|format_datetime('medium', 'short') }}
                </small>

                <p>{{ comment.text }}</p>
            {% endfor %}
        {% else %}
            <div>No comments have been posted yet for this conference.</div>
        {% endif %}
    {% endblock %}

У цьому шаблоні ми використовуємо позначення ``|``, щоб викликати *фільтри* Twig. Фільтр перетворює значення. Наприклад, ``comments|length`` повертає кількість коментарів, а ``comment.createdAt|format_datetime('medium', 'short')`` форматує дату в читабельне представлення.

Спробуйте відкрити сторінку "першої" конференції за шляхом ``/conference/1`` ​​і зверніть увагу на наступну помилку:

.. figure:: screenshots/intl-twig-error.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

Причиною помилки є використання фільтра ``format_datetime``, оскільки він не є частиною ядра Twig. Повідомлення про помилку дає вам підказку про те, який пакет слід встановити, щоб усунути проблему:

.. code-block:: terminal

    $ symfony composer req "twig/intl-extra:^3"

Тепер сторінка працює належним чином.

Перелінкування сторінок
---------------------------------------------

.. index::
    single: Twig;Link
    single: Link

Останнім кроком до завершення першої версії користувацького інтерфейсу є створення посилань на конференції з головної сторінки:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -7,5 +7,8 @@

         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
    +        <p>
    +            <a href="/conference/{{ conference.id }}">View</a>
    +        </p>
         {% endfor %}
     {% endblock %}

Але жорстке визначення шляху є поганою ідеєю з кількох причин. Найважливіша з них полягає в тому, що якщо ви зміните шлях (наприклад, з ``/conference/{id}`` на ``/conferences/{id}``), всі посилання доведеться змінювати вручну.

.. index::
    single: Twig;path

Натомість використовуйте *функцію* Twig ``path()`` і  *ім'я маршруту*:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="/conference/{{ conference.id }}">View</a>
    +            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}

Функція ``path()`` генерує шлях до сторінки, використовуючи ім'я її маршруту. Значення параметрів маршруту передаються у вигляді об'єкта Twig.

Пагінація коментарів
---------------------------------------

.. index::
    single: Doctrine;Paginator
    single: Paginator

З тисячами відвідувачів ми можемо очікувати досить багато коментарів. Якщо ми будемо відображати їх на одній сторінці, вона буде дуже швидко рости.

Створіть метод ``getCommentPaginator()`` у репозиторії коментарів, який повертає *пагінатор* коментарів на основі конференції й зміщення (відносно початку):

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -3,19 +3,37 @@
     namespace App\Repository;

     use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
      * @extends ServiceEntityRepository<Comment>
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    public const COMMENTS_PER_PAGE = 2;
    +
         public function __construct(ManagerRegistry $registry)
         {
             parent::__construct($registry, Comment::class);
         }

    +    public function getCommentPaginator(Conference $conference, int $offset): Paginator
    +    {
    +        $query = $this->createQueryBuilder('c')
    +            ->andWhere('c.conference = :conference')
    +            ->setParameter('conference', $conference)
    +            ->orderBy('c.createdAt', 'DESC')
    +            ->setMaxResults(self::COMMENTS_PER_PAGE)
    +            ->setFirstResult($offset)
    +            ->getQuery()
    +        ;
    +
    +        return new Paginator($query);
    +    }
    +
         //    /**
         //     * @return Comment[] Returns an array of Comment objects
         //     */

Ми встановили максимальну кількість коментарів на сторінці рівним 2, щоб полегшити тестування.

Щоб керувати пагінацією в шаблоні, передайте пагінатор Doctrine замість колекції Doctrine в Twig:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,6 +8,7 @@ use App\Repository\ConferenceRepository;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
     use Twig\Environment;

    @@ -22,11 +23,16 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
    +        $offset = max(0, $offset);
    +        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
    +
             return new Response($twig->render('conference/show.html.twig', [
                 'conference' => $conference,
    -            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +            'comments' => $paginator,
    +            'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
    +            'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
             ]));
         }
     }

Атрибут ``#[MapQueryParameter]`` зіставляє параметр ``offset`` з рядка запиту з аргументом контролера ``$offset`` зі значенням за замовчуванням ``0``, якщо його не встановлено. Оскільки offset надходить від клієнта, ми обмежуємо його, щоб уникнути від'ємних значень.

Зміщення ``previous`` і ``next`` обчислюються на основі всієї інформації, яку ми отримали з пагінатора.

.. index::
    single: Twig;if

Нарешті, оновіть шаблон, щоб додати посилання на наступну та попередню сторінки:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -6,6 +6,8 @@
         <h2>{{ conference }} Conference</h2>

         {% if comments|length > 0 %}
    +        <div>There are {{ comments|length }} comments.</div>
    +
             {% for comment in comments %}
                 {% if comment.photofilename %}
                     <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
    @@ -18,6 +20,13 @@

                 <p>{{ comment.text }}</p>
             {% endfor %}
    +
    +        {% if previous >= 0 %}
    +            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +        {% endif %}
    +        {% if next < comments|length %}
    +            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +        {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}

Тепер ви зможете переміщуватися по коментарях через посилання  "Previous" та "Next":

.. figure:: screenshots/pagination-next.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

.. figure:: screenshots/pagination-previous.png
    :alt: /conference/1?offset=2
    :align: center
    :figclass: with-browser

Рефакторинг контролера
-------------------------------------------

Можливо ви помітили, що обидва методи в ``ConferenceController`` отримують середовище Twig у якості аргументу. Замість того щоб додавати його в кожен метод, використовуймо допоміжний метод ``render()``, що надається наслідуваним класом:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,29 +9,28 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
    -use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
    +    public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($twig->render('conference/index.html.twig', [
    +        return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]));
    +        ]);
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    -        return new Response($twig->render('conference/show.html.twig', [
    +        return $this->render('conference/show.html.twig', [
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    -        ]));
    +        ]);
         }
     }

.. sidebar:: Йдемо далі

    * `Документація по Twig`_;

    * `Створення й використання шаблонів`_ у застосунках Symfony;

    * `Навчальний посібник SymfonyCasts: Twig`_;

    * `Функції та фільтри Twig, які доступні лише в Symfony`_;

    * `Базовий контролер AbstractController`_.

.. _`Twig`: https://twig.symfony.com/
.. _`Документація по Twig`: https://twig.symfony.com/doc/3.x/
.. _`Створення й використання шаблонів`: https://symfony.com/doc/current/templates.html
.. _`Навчальний посібник SymfonyCasts: Twig`: https://symfonycasts.com/screencast/symfony/twig-recipe
.. _`Функції та фільтри Twig, які доступні лише в Symfony`: https://symfony.com/doc/current/reference/twig_reference.html
.. _`Базовий контролер AbstractController`: https://symfony.com/doc/current/controller.html#the-base-controller-classes-services
