Создание контроллера
=======================================

.. index::
    single: Controller
    single: Routing;Route

Наш проект гостевой книги уже работает в продакшене, однако мы немного схитрили. У проекта пока ещё нет ни одной веб-страницы, а на главной странице по-прежнему отображается ошибка 404. Давайте исправим это.

Когда приходит HTTP-запрос, например, на главную страницу (``http://localhost:8000/``), Symfony пытается найти *маршрут* , соответствующий *пути запроса* (``/`` в данном случае). *Маршрут* — это связующее звено между путём запроса и *callback-функцией PHP*, которая создаёт *ответ* HTTP для этого запроса.

Эти callback-функции PHP называются "контроллерами". В Symfony большинство контроллеров реализовано в виде PHP-классов. Конечно, вы можете вручную создать такой класс, но для быстроты разработки давайте посмотрим, как Symfony в этом может нам помочь.

Ускорение разработки с помощью бандла Maker
----------------------------------------------------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Для лёгкой генерации контроллеров мы можем использовать пакет ``symfony/maker-bundle``:

.. code-block:: bash

    $ symfony composer req maker --dev

Так как бандл Maker полезен лишь в процессе разработки, не забудьте указать флаг ``--dev``, чтобы пакет не загружался при развёртывании приложения в продакшене.

Бандл Maker поможет вам сгенерировать множество различных классов. Мы будем использовать его на протяжении всей книги. Каждый "генератор" определён в отдельной команде, при этом все команды принадлежат одному и тому же пространству имён команды ``make``.

.. index::
    single: Command;list

Компонент Symfony Console имеет встроенную команду ``list``, которая выводит список всех команд в указанном пространстве имен; используйте её, чтобы посмотреть все доступные генераторы бандла Maker:

.. code-block:: bash
    :class: ignore

    $ symfony console list make

Выбор формата конфигурации
--------------------------------------------------

Перед созданием нашего первого контроллера в проекте, нам необходимо определиться с используемыми форматами для написания конфигураций. По умолчанию Symfony поддерживает YAML, XML, PHP и аннотации.

*YAML* идеален для *конфигурации пакетов*. Этот формат будет использоваться в файлах директории ``config/``. Зачастую, когда вы устанавливаете новый пакет, рецепт этого пакета добавляет новый файл с расширением ``.yaml`` в эту директорию.

Для *конфигурации, связанной с PHP-кодом*, лучше всего подойдут *аннотации*, поскольку они указываются рядом с кодом. Сейчас объясню на примере. Когда приходит запрос, определённая конфигурация должна передать Symfony, что текущий путь запроса должен обрабатываться соответствующим контроллером (то есть PHP-классом). При использовании конфигурационных форматов на YAML, XML или PHP потребуется включать два файла (сам конфигурационный файл и PHP-файл контроллера). В то же время с помощью аннотаций можно всё сконфигурировать непосредственно в самом классе контроллера.

Для работы с аннотациями нужно добавить ещё одну зависимость:

.. code-block:: bash

    $ symfony composer req annotations

Вам может быть интересно узнать, какой именно пакет нужно установить, чтобы добавить поддержку той или иной функциональной возможности? Как правило, вам не нужно этого знать. Поскольку в большинстве случаев Symfony показывает отсутствующий пакет в сообщении об ошибке. К примеру, выполнение команды ``symfony make:controller`` без установленного пакета ``annotations`` выбросит исключение с подсказкой о том, какой пакет следует установить.

Генерация контроллера
-----------------------------------------

.. index::
    single: Command;make:controller

Создайте свой первый *контроллер* с помощью команды ``make:controller``:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;@Route

Команда создает класс ``ConferenceController`` в директории ``src/Controller/``. Сгенерированный класс содержит немного шаблонного кода, который может быть изменён:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 10

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class ConferenceController extends AbstractController
    {
        /**
         * @Route("/conference", name="conference")
         */
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

Аннотация ``@Route("/conference", name="conference")`` — это именно то, что делает метод ``index()`` контроллером (конфигурация находится рядом с кодом, который она настраивает).

При переходе в браузере по пути ``/conference`` выполняется контроллер, который возвращает HTTP-ответ.

Настройте маршрут так, чтобы он соответствовал главной странице:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,7 +9,7 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/conference", name="conference")
    +     * @Route("/", name="homepage")
          */
         public function index(): Response
         {

Параметр маршрута ``name`` пригодится нам, когда понадобится получить ссылку на главную страницу. Таким образом, вместо жёстко заданного в коде пути ``/``, мы будем использовать имя маршрута.

Теперь вернём обычный HTML-код вместо шаблона по умолчанию:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -13,8 +13,13 @@ class ConferenceController extends AbstractController
          */
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

Перезагрузите главную страницу в браузере:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Основная задача контроллера — вернуть объект ``Response`` в качестве ответа на HTTP-запрос.

.. _easter-egg:

Добавление пасхального яйца
----------------------------------------------------

Для демонстрации того, как в ответе можно использовать информацию из запроса, давайте добавим небольшое `пасхальное яйцо`_. При переходе на главную страницу, если в строке запроса мы находим что-то типа ``?hello=Fabien``, то показываем приветственное сообщение.

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
         /**
          * @Route("/", name="homepage")
          */
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

Посмотреть данные запроса в Symfony можно через объект ``Request``. Если в объявлении типа для аргумента контроллера указать объект запроса, Symfony автоматически передаст его вам. Далее используем этот объект, чтобы получить параметр ``name`` из строки запроса и добавляем его в заголовок ``<h1>``.

Чтобы увидеть получившийся результат, сначала в браузере перейдите на главную страницу (``/``), а после по пути ``/?hello=Fabien``.

.. note::

    Обратите внимание на вызов функции ``htmlspecialchars()``, который нужен, чтобы защититься от XSS-атак. Когда мы начнём использовать шаблонизатор, он автоматически защитит нас от подобных проблем с безопасностью.

Также отмечу, что имя можно сделать частью URL-адреса:

.. code-block:: diff
    :emphasize-lines: 8,11
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,13 +9,19 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/", name="homepage")
    +     * @Route("/hello/{name}", name="homepage")
          */
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

Часть маршрута вроде ``{name}`` называется *параметром маршрута* — он работает так же, как и знак подстановки. Теперь можно перейти по пути ``/hello``, а затем на ``/hello/Fabien`` — результат будет одинаковый. *Значение* параметра ``{name}`` можно получить, добавив *одноимённый* аргумент в контроллер (то есть ``$name``).

.. sidebar:: Двигаемся дальше

    * Система `маршрутизации <https://symfony.com/doc/current/routing.html>`_ в Symfony;

    * `Обучающий видеоролик по маршрутам, контроллерам и страницам на SymfonyCasts <https://symfonycasts.com/screencast/symfony/route-controller>`_;

    * `Аннотации <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_ в PHP;

    * Компонент `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_;

    * Методика атаки через `межсайтовый скриптинг (Cross-Site Scripting или XSS) <https://owasp.org/www-community/attacks/xss/>`_;

    * `Шпаргалка по маршрутизации в Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`пасхальное яйцо`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
