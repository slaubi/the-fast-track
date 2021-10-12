Створення контролера
=======================================

.. index::
    single: Controller
    single: Routing;Route

Наш проект гостьової книги вже працює на продакшн-сервері, але ми трохи схитрували. Проект поки не має жодної веб-сторінки. Головною сторінкою служить нудна сторінка помилок 404. Виправімо це.

Коли надходить HTTP запит, наприклад, для головної сторінки (``http://localhost:8000/``), Symfony намагається знайти *маршрут*, який підходить до *шляху запиту* (тут ``/``). *Маршрут* (*route*) є сполучною ланкою між шляхом запиту та *PHP-callable* функцією, яка створює HTTP *відповідь* для цього запиту.

Ці "callable" називаються "контролерами". У Symfony більшість контролерів реалізовані у вигляді PHP-класів. Ви можете створити такий клас вручну, але ж нам подобається рухатися швидко, подивімося, як Symfony може нам допомогти.

Байдикування з бандлом Maker
------------------------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Щоб легко генерувати контролери, ми можемо використовувати пакет ``symfony/maker-bundle``:

.. code-block:: bash

    $ symfony composer req maker --dev

Оскільки бандл Maker корисний тільки під час розробки, не забудьте додати прапорець ``--dev``, щоб уникнути його встановлення у продакшн.

Бандл Maker допоможе вам згенерувати безліч різних класів. Ми будемо використовувати його постійно, впродовж цієї книги. Кожен "генератор" визначено в команді, при цьому всі команди є частиною простору імен ``make``.

.. index::
    single: Command;list

Вбудована в Symfony Console команда ``list`` перелічує всі доступні команди в рамках заданого простору імен; використовуйте її для перегляду всіх генераторів, що надаються бандлом Maker:

.. code-block:: bash
    :class: ignore

    $ symfony console list make

Вибір формату конфігурації
--------------------------------------------------

Перед створенням першого контролера у проекті, нам необхідно вирішити, які формати конфігурації ми хочемо використовувати. За замовчуванням Symfony підтримує YAML, XML, PHP та анотації.

Для *пакетів, які пов'язані з конфігурацією*, *YAML* є найкращим вибором. Це формат, який використовується в каталозі ``config/``. Часто, коли ви встановлюєте новий пакет, рецепт цього пакета додає новий файл, що закінчується на ``.yaml``, у цей каталог.

Для *конфігурації, пов'язаної з кодом PHP*, *анотації* є кращим вибором, оскільки вони визначені поруч з кодом. Поясню на прикладі. Коли надходить запит, певні елементи конфігурації мають сповістити Symfony про те, що шлях запиту має бути оброблений конкретним контролером (класом PHP). При використанні форматів конфігурації YAML, XML або PHP залучаються два файли (файл конфігурації та файл контролера PHP). При використанні анотацій, налаштування проводиться безпосередньо в класі контролера.

Для роботи з анотаціями потрібно додати ще одну залежність:

.. code-block:: bash

    $ symfony composer req annotations

Вам може бути цікаво, як ви можете здогадатися про ім'я пакета, який потрібно встановити для тої чи іншої можливості? Зазвичай вам не потрібно цього знати. У багатьох випадках Symfony містить відомості про пакет, який потрібно встановити, у своїх повідомленнях про помилки. Наприклад, запуск ``symfony make:controller`` без пакета ``annotations`` закінчився б винятком, що містить підказку про необхідність встановлення потрібного пакету.

Генерування контролера
-------------------------------------------

.. index::
    single: Command;make:controller

Створіть ваш перший *Контролер*, за допомогою команди ``make:controller``:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;Route

Ця команда створює клас ``ConferenceController`` у каталозі ``src/Controller/``. Згенерований клас складається з шаблонного коду, готового до більш ретельного налаштування.

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

Анотація ``#[Route('/conference', name: 'conference')]`` є тим, що робить метод ``index()`` контролером (конфігурація знаходиться поруч із кодом, який вона налаштовує).

Коли ви переходите за посиланням ``/conference`` в браузері, виконується контролер та повертається відповідь.

Налаштуйте маршрут так, щоб він відповідав головній сторінці:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

``Ім'я`` (``name``) маршруту буде корисним, коли ми захочемо отримати посилання на головну сторінку в коді. Замість того, щоб "жорстко" вказувати шлях маршруту ``/``, ми будемо використовувати його ім'я.

Замість відмальованої сторінки за замовчуванням, повернімо простий HTML:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +<html>
    +    <body>
    +        <img src="/images/under-construction.gif" />
    +    </body>
    +</html>
    +EOF
    +        );
         }
     }

Оновіть веб-сторінку в браузері:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Основний обов'язок контролера — повернути ``HTTP-відповідь`` на запит.

.. _easter-egg:

Додавання пасхалки
-----------------------------------

Щоб продемонструвати те, як відповідь може використовувати інформацію із запиту, додаймо невеличку `пасхалку`_. Щоразу, коли шлях головної сторінки міститиме рядок запиту, типу ``?hello=Fabien`` — додаймо трохи тексту, щоб привітати цю людину:

.. code-block:: diff
    :emphasize-lines: 16
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,6 +3,7 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

    @@ -11,11 +12,17 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Symfony надає дані запиту через об'єкт ``Request``. Коли Symfony бачить аргумент контролера цього типу, він автоматично знає, як передати його вам. Ми можемо використовувати його, щоб отримати елемент ``name`` з рядка запиту і додати заголовок ``<h1>``.

Спробуйте перейти до ``/``, потім ``/?hello=Fabien``, щоб побачити різницю.

.. note::

    Зверніть увагу на виклик функції ``htmlspecialchars()`` для уникнення проблем з XSS. Це те, що за нас буде зроблено автоматично, коли ми перейдемо на правильний шаблонізатор.

Ми також могли б зробити ім’я частиною URL-адреси:

.. code-block:: diff
    :emphasize-lines: 10,11
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,13 +9,19 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    +    #[Route('/hello/{name}', name: 'homepage')]
    -    public function index(): Response
    +    public function index(string $name = ''): Response
         {
    +        $greet = '';
    +        if ($name) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Частина маршруту ``{name}`` є динамічним *параметром маршруту* — він працює як символ підстановки. Тепер ви можете перейти до ``/hello``, а потім до ``/hello/Fabien`` в браузері, щоб отримати ті ж результати, що і раніше. Ви можете отримати *значення* параметра ``{name}``, додавши аргумент контролера з тим же *ім'ям* (*name*). Отже, ``$name``.

.. sidebar:: Йдемо далі

    * Система `маршрутизації <https://symfony.com/doc/current/routing.html>`_ в Symfony;

    * `Навчальний посібник SymfonyCasts: маршрути, контролери та сторінки <https://symfonycasts.com/screencast/symfony/route-controller>`_;

    * `Анотації <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_ у PHP;

    * Компонент `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_;

    * Атаки на систему безпеки: `XSS (міжсайтовий скриптинг) <https://owasp.org/www-community/attacks/xss/>`_;

    * `Шпаргалка по маршрутизації в Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`пасхалку`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
