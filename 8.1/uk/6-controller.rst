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

Щоб легко генерувати контролери, ми можемо використовувати бандл ``symfony/maker-bundle``, який було встановлено як частину пакета ``webapp``.

Бандл Maker допоможе вам згенерувати безліч різних класів. Ми будемо використовувати його постійно, впродовж цієї книги. Кожен "генератор" визначено в команді, при цьому всі команди є частиною простору імен ``make``.

.. index::
    single: Command;list

Вбудована в Symfony Console команда ``list`` перелічує всі доступні команди в рамках заданого простору імен; використовуйте її для перегляду всіх генераторів, що надаються бандлом Maker:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

Вибір формату конфігурації
--------------------------------------------------

Перш ніж створити перший контролер проекту, нам потрібно визначитися з форматами конфігурації, які ми хочемо використовувати. Symfony підтримує YAML, XML, PHP і PHP атрибути з коробки.

Для *пакетів, які пов'язані з конфігурацією*, *YAML* є найкращим вибором. Це формат, який використовується в каталозі ``config/``. Часто, коли ви встановлюєте новий пакет, рецепт цього пакета додає новий файл, що закінчується на ``.yaml``, у цей каталог.

Для *конфігурації, пов'язаної з кодом PHP*, *атрибути* є кращим вибором, оскільки вони визначені поруч із кодом. Поясню на прикладі. Коли надходить запит, певні елементи конфігурації мають сповістити Symfony про те, що шлях запиту має бути оброблений конкретним контролером (класом PHP). При використанні форматів конфігурації YAML, XML або PHP залучаються два файли (файл конфігурації та файл контролера PHP). При використанні атрибутів налаштування проводиться безпосередньо в класі контролера.

Вам може бути цікаво, як ви можете здогадатися про ім'я пакета, який потрібно встановити для тої чи іншої можливості? Зазвичай вам не потрібно цього знати. У багатьох випадках Symfony містить відомості про пакет, який потрібно встановити, у своїх повідомленнях про помилки. Наприклад, виконання ``symfony console make:message`` без пакета ``messenger`` закінчився б винятком, який містив би підказку щодо встановлення правильного пакета.

Генерування контролера
-------------------------------------------

.. index::
    single: Command;make:controller

Створіть ваш перший *Контролер*, за допомогою команди ``make:controller``:

.. code-block:: terminal

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

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
        #[Route('/conference', name: 'app_conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

Атрибут ``#[Route('/conference', name: 'app_conference')]`` є тим, що робить метод ``index()`` контролером (конфігурація знаходиться поруч із кодом, який він налаштовує).

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
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

``Ім'я`` (``name``) маршруту буде корисним, коли ми захочемо отримати посилання на головну сторінку в коді. Замість того щоб "жорстко" вказувати шлях маршруту ``/``, ми будемо використовувати його ім'я.

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
    +            <html>
    +                <body>
    +                    <img src="/images/under-construction.gif" />
    +                </body>
    +            </html>
    +            EOF
    +        );
         }
     }

Оновіть веб-сторінку в браузері:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Основний обов'язок контролера — повернути ``HTTP-відповідь`` на запит.

Оскільки решта розділу присвячена коду, який ми не будемо зберігати, зафіксуймо наші зміни:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

Додавання пасхалки
-----------------------------------

Щоб продемонструвати те, як відповідь може використовувати інформацію із запиту, додаймо невеличку `пасхалку`_. Щоразу, коли шлях головної сторінки міститиме рядок запиту, типу ``?hello=Fabien`` — додаймо трохи тексту, щоб привітати цю людину:

.. code-block:: diff
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
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
    +                    $greet
                         <img src="/images/under-construction.gif" />
                     </body>
                 </html>

Symfony надає дані запиту через об'єкт ``Request``. Коли Symfony бачить аргумент контролера цього типу, він автоматично знає, як передати його вам. Ми можемо використовувати його, щоб отримати елемент ``name`` з рядка запиту і додати заголовок ``<h1>``.

Спробуйте перейти до ``/``, потім ``/?hello=Fabien``, щоб побачити різницю.

.. note::

    Зверніть увагу на виклик функції ``htmlspecialchars()`` для уникнення проблем з XSS. Це те, що за нас буде зроблено автоматично, коли ми перейдемо на правильний шаблонізатор.

Ми також могли б зробити ім’я частиною URL-адреси:

.. code-block:: diff

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    -    public function index(Request $request): Response
    +    #[Route('/hello/{name}', name: 'homepage')]
    +    public function index(string $name = ''): Response
         {
             $greet = '';
    -        if ($name = $request->query->get('hello')) {
    +        if ($name) {
                 $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
             }

Частина маршруту ``{name}`` є динамічним *параметром маршруту* — він працює як символ підстановки. Тепер ви можете перейти до ``/hello``, а потім до ``/hello/Fabien`` в браузері, щоб отримати ті ж результати, що і раніше. Ви можете отримати *значення* параметра ``{name}``, додавши аргумент контролера з тим же *ім'ям* (*name*). Отже, ``$name``.

Поверніть зміни, які ми щойно внесли:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

Налагодження змінних
---------------------------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Відмінним помічником для налагодження є функція Symfony ``dump()``. Вона завжди доступна і дозволяє налагоджувати складні змінні в приємному інтерактивному форматі.

Тимчасово змініть ``src/Controller/ConferenceController.php``, щоб налагодити об’єкт запиту:

.. code-block:: diff
    :emphasize-lines: 17

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        dump($request);
    +
             return new Response(<<<EOF
                 <html>
                     <body>

Під час оновлення сторінки зверніть увагу на новий значок "target" на панелі інструментів; він дозволяє оглянути дані налагодження. Натисніть на нього, щоб отримати доступ до повної сторінки зі спрощеною навігацією:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Поверніть зміни, які ми щойно внесли:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: Йдемо далі

    * Система `маршрутизації`_ в Symfony;

    * `Навчальний посібник SymfonyCasts: маршрути, контролери та сторінки`_;

    * `Атрибути у PHP`_;

    * Компонент `HttpFoundation`_;

    * Атаки на систему безпеки: `XSS (міжсайтовий скриптинг)`_;

    * `Шпаргалка по маршрутизації в Symfony`_.

.. _`пасхалку`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`маршрутизації`: https://symfony.com/doc/current/routing.html
.. _`Навчальний посібник SymfonyCasts: маршрути, контролери та сторінки`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`Атрибути у PHP`: https://www.php.net/attributes
.. _`HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (міжсайтовий скриптинг)`: https://owasp.org/www-community/attacks/xss/
.. _`Шпаргалка по маршрутизації в Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
