Повышение производительности с помощью кеширования
================================================================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

По мере роста популярности могут возникнуть проблемы с производительностью. Вот только пара типичных примеров: отсутствие индексов в базе данных или огромное количество SQL-запросов на страницу. Вам нечего боятся при пустой базе данных, но с ростом трафика и объема данных могут начаться проблемы.

Добавление заголовков кеширования HTTP
---------------------------------------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

HTTP-кеширование — отличный способ увеличить производительность для пользователей с минимальными усилиями. Для большей производительности в продакшене используйте кеширующий обратный прокси-сервер и пограничный `CDN`_.

Давайте закешируем главную страницу на один час:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -29,7 +29,7 @@ final class ConferenceController extends AbstractController
         {
             return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Метод ``setSharedMaxAge()`` устанавливает срок действия кеша для обратных прокси-серверов. Используйте метод ``setMaxAge()``, чтобы управлять кешем браузера. Время указывается в секундах (1 час = 60 минут = 3600 секунд).

Кеширование страницы конференции является менее тривиальной задачей, поскольку данные на ней более динамичны. В любой момент любой желающий может добавить комментарий, и никто не хочет ждать целый час, чтобы его увидеть на сайте. В таких случаях используйте *валидацию кеширования HTTP*.

Активация ядра HTTP-кеширования в Symfony
------------------------------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Для тестирования стратегии HTTP-кеширования включите обратный прокси-сервер Symfony, но только в среде "development" (для среды "production" мы будем использовать "более надёжное" решение):

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Являясь полноценным обратным прокси-сервером HTTP, он дополнительно (с помощью класса ``HttpCache``) добавляет полезную отладочную информацию в виде HTTP-заголовков. Она может сильно помочь в проверке заголовков кеширования, которые мы установили.

Проверим, как работает кеширование на главной странице:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

Для самого первого запроса кеширующий сервер говорит нам, что ответ на запрос не был закеширован (кеш-промах, или ``miss``) и что он сохранил полученный ответ (``store``) для будущих запросов. Посмотрите на заголовок ``cache-control``, чтобы увидеть настроенную стратегию кеширования.

Для последующих запросов ответ будет закеширован (при этом ``age`` также обновился):

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

Кеширование SQL-запросов при помощи ESI
-------------------------------------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Обработчик ``TwigEventSubscriber`` внедряет глобальную переменную в Twig для всех объектов конференции. Это происходит для каждой отдельной страницы сайта. Вероятно, это отличное место для оптимизации.

Вы не добавляете новые конференции каждый день, однако код запрашивает одни и те же данные из базы снова и снова.

Возможно, нам стоит закешировать названия конференций и слаги с помощью Symfony Cache, но там, где это возможно, я предпочёл бы использовать HTTP-кеширование.

Когда вам нужно закешировать фрагмент страницы, не загружайте его в текущем HTTP-запросе, а вместо этого создайте *подзапрос* с ним. *ESI* идеально подходит для решения такой задачи. ESI — это способ вставить результат одного HTTP-запроса в другой.

Создайте контроллер, возвращающий только HTML-фрагмент с конференциями:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -32,6 +32,14 @@ final class ConferenceController extends AbstractController
             ])->setSharedMaxAge(3600);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,

Создайте соответствующий шаблон:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Посетите ``/conference_header``, чтобы проверить, что всё работает.

.. index::
    single: Twig;render
    single: Twig;path

Раскроем тайну фокуса! Обновите шаблон Twig, чтобы вызывать только что созданный контроллер:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

И *вуаля*. Перезагрузите страницу в браузере — на сайте по-прежнему будет показано то же самое.

.. tip::

    Воспользуйтесь панелью "Request / Response" в профилировщике Symfony, чтобы узнать больше об основном запросе и его подзапросах.

Теперь каждый раз, когда вы открываете страницу в браузере, выполняются два HTTP-запроса: один с конференциями для шапки сайта, другой — для главной страницы. Вы только что ухудшили производительность. Поздравляю!

HTTP-запрос для получения списка конференций сейчас выполняется изнутри в Symfony, поэтому HTTP-вызова нет. Помимо всего, это означает, что мы не сможем извлечь выгоду от заголовков HTTP-кеширования.

Поэтому нужно превратить этот запрос в "настоящий" HTTP с помощью ESI.

Во-первых, включите поддержку ESI:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Далее замените ``render`` на ``render_esi``:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Symfony автоматически активирует поддержку ESI, если обратный прокси-сервер умеет работать с ESI (в противном случае переходит к синхронной обработке подзапросов).

Поскольку обратный прокси-сервер Symfony поддерживает ESI, давайте проверим его логи (сначала очистите кеш, как описано в следующем разделе):

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

Обновите страницу несколько раз: страница по адресу ``/`` кешируется, а по ``/conference_header`` — нет. Неплохой результат: вся страница кешируется, при этом динамическая часть всё ещё присутствует.

Однако это не то, что нам нужно. Давайте закешируем страницу с конференциями на один час, независимо от всего остального:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -37,7 +37,7 @@ final class ConferenceController extends AbstractController
         {
             return $this->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Кеширование теперь включено для обоих запросов:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

Заголовок ``x-symfony-cache`` содержит два элемента: основной запрос на ``/`` и подзапрос (``conference_header`` через ESI). Ответы на оба запроса находятся в кеше (``fresh``).

Стратегия кеширования может различаться между главной страницей и теми, которые загружаются через ESI. Допустим, у нас есть редко обновляемая страница "О нас", то мы можем закешировать её на неделю, и при этом по-прежнему обновлять страницу с конференциями каждый час.

Удалите обработчик, так как он нам больше не нужен:

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Очистка HTTP-кеша для тестирования
------------------------------------------------------------

Тестирование сайта в браузере или с помощью автотестов становится немного сложнее при наличии слоя кеширования.

Вы можете вручную очистить весь HTTP-кеш, если удалите директорию ``var/cache/dev/http_cache/``:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

Такой подход неудобен и не эффективен, если вам нужно инвалидировать кеш только определённых URL-адресов или интегрировать инвалидацию кеша в функциональные тесты. Давайте добавим HTTP-маршрут, который будет доступен только для администратора и при помощи которого можно будет сбросить кеш для некоторых URL-адресов:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

Новый контроллер обрабатывает только HTTP-метод ``PURGE``. Этот метод не входит в стандарт HTTP, но широко используется для инвалидации кеша.

По умолчанию параметры маршрута не могут содержать сами символы ``/``, так как они разделяют сегменты URL-адреса. Но вы можете обойти это ограничение для последнего параметра маршрута, как это было сделано в ``uri``, если зададите для него нужное вам выражение (``.*``).

Получение объекта ``HttpCache`` выглядит немного странным — мы используем анонимный класс, так как получить доступ к "настоящему" невозможно. Объект ``HttpCache`` оборачивает ядро фреймворка, которое ничего не знает о слое кеширования, но так и должно быть.

Инвалидируйте главную страницу и страницу с конференциями, выполнив следующие cURL-команды:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

Подкоманда ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` возвращает текущий URL-адрес локального веб-сервера.

.. note::

    Контроллер не имеет имени маршрута, так как он никогда не будет использован в коде.

Группировка схожих маршрутов по префиксу
----------------------------------------------------------------------------

.. index::
    single: Attributes;Route

Два маршрута в административном контроллере имеют одинаковый префикс ``/admin``. Вместо того, чтобы повторять его для всех маршрутов, улучшим маршруты таким образом, чтобы префикс был определён в самом классе:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

Кеширование операций с большим потреблением ресурсов ЦПУ/памяти
-----------------------------------------------------------------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

У нас на сайте нет алгоритмов, интенсивно использующих ЦПУ или память. Чтобы рассмотреть *локальное кеширование*, давайте создадим команду, которая будет отображать шаг, над которым мы сейчас работаем (если точнее, то мы выведем имя Git-тега, прикреплённого к текущему коммиту).

Компонент Symfony Process позволяет выполнить команду и получить результат (стандартный вывод и вывод ошибок).

Реализуйте команду:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand extends Command
    {
        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Вы можете выполнить ``make:command`` для создания команды:

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Как насчёт того, чтобы закешировать результат на несколько минут? Установите Symfony-компонент Cache.

А затем оберните код логикой кеширования:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -7,15 +7,27 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand extends Command
     {
    +    public function __construct(
    +         private CacheInterface $cache,
    +    ) {
    +        parent::__construct();
    +    }
    +
         protected function execute(InputInterface $input, OutputInterface $output): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $this->cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

Теперь процесс вызывается только в том случае, если элемент ``app.current_step`` отсутствует в кеше.

Профилирование и сравнение производительности
---------------------------------------------------------------------------------------

Никогда не используйте кеширование вслепую. Имейте в виду, что использование кеширования добавляет ещё один уровень сложности. Сложно предсказать, что будет работать быстро, а что — медленно. В итоге вы можете оказаться в ситуации, когда кеширование замедлит работу вашего приложения.

Всегда измеряйте результат от использования кеширования с помощью инструментов профилировщика, например, `Blackfire`_.

Обратитесь к шагу про производительность, чтобы узнать больше о том, как использовать Blackfire для тестирования перед развёртыванием.

Настройка кеширующего обратного прокси-сервера в продакшене
----------------------------------------------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Platform.sh;Varnish
    single: Varnish

Вместо использования обратного прокси-сервера Symfony в продакшене, мы будем использовать "более надёжный" обратный прокси-сервер Varnish.

Добавьте Varnish в сервисы Platform.sh:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform/services.yaml
    +++ w/.platform/services.yaml
    @@ -4,3 +4,11 @@ database:
         disk: 1024


    +varnish:
    +    type: varnish:6.0
    +    relationships:
    +        application: 'app:http'
    +    configuration:
    +        vcl: !include
    +            type: string
    +            path: config.vcl

.. index::
    single: Platform.sh;Routes

Используйте Varnish в качестве основной точки входа в маршруты:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform/routes.yaml
    +++ w/.platform/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Наконец, создайте файл ``config.vcl`` для конфигурации Varnish:

.. code-block:: vcl
    :caption: .platform/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Включение поддержки ESI в Varnish
----------------------------------------------------

Поддержка ESI в Varnish должна быть включена явно для каждого запроса. Чтобы сделать это для всех запросов сразу, Symfony использует стандартные заголовки ``Surrogate-Capability`` и ``Surrogate-Control``:

.. code-block:: vcl
    :caption: .platform/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Инвалидация кеша Varnish
---------------------------------------

Инвалидация кеша в продакшене, скорее всего, никогда не понадобится, кроме экстренных случаев и, возможно, только в ветках, отличных от ``master``. Если вам приходится часто чистить кеш, то обычно это означает, что стратегия кеширования должна быть скорректирована (путём уменьшения TTL или с помощью использования стратегии валидации кеша вместо истечения срока действия).

В любом случае, давайте посмотрим, как сконфигурировать Varnish для инвалидации кеша:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform/config.vcl
    +++ w/.platform/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

Скорее всего, в реальности вы разрешите очистку кеша только с определённых IP-адресов, как об этом описано в `документации Varnish`_.

Теперь очистите кеш для некоторых URL-адресов:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

URL-адреса выглядит немного необычно, потому что команда ``env:url`` возвращает адреса, которые уже заканчиваются символом ``/``.

.. sidebar:: Двигаемся дальше

    * Глобальная облачная платформа `Cloudflare`_;

    * `Документация по Varnish`_;

    * `Спецификация ESI`_ и `ресурсы по ESI`_;

    * `Модель валидации HTTP-кеширования`_;

    * `HTTP-кеширование в Platform.sh`_.

.. _`Blackfire`: https://blackfire.io/
.. _`документации Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Документация по Varnish`: https://varnish-cache.org/docs/index.html
.. _`Спецификация ESI`: https://www.w3.org/TR/esi-lang
.. _`ресурсы по ESI`: https://www.akamai.com/us/en/support/esi.jsp
.. _`Модель валидации HTTP-кеширования`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP-кеширование в Platform.sh`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
