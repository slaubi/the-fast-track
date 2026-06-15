Кешування для підвищення продуктивності
===========================================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Проблеми з продуктивністю можуть виникнути разом зі зростанням популярності. Кілька типових прикладів: відсутність індексів бази даних або безліч SQL-запитів на сторінку. У вас не буде жодних проблем із порожньою базою даних, але зі збільшенням кількості трафіку та дедалі більшим обсягом даних, в якийсь момент, вони можуть виникнути.

Додавання заголовків HTTP-кешу
-----------------------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Використання стратегій HTTP-кешування є відмінним способом досягнення максимальної продуктивності для кінцевих користувачів без особливих зусиль. Додайте зворотний проксі-кеш у продакшн, щоб увімкнути кешування і використовуйте `CDN`_  для досягнення ще кращих результатів продуктивності.

Закешуймо головну сторінку на одну годину:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -30,7 +30,7 @@ class ConferenceController extends AbstractController
         {
             return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Метод ``setSharedMaxAge()`` встановлює термін дії кешу для зворотних проксі. Використовуйте ``setMaxAge()``, щоб контролювати кеш браузера. Час встановлюється у секундах (1 година = 60 хвилин = 3600 секунд).

Кешувати сторінку конференції складніше, оскільки вона більш динамічна. В будь-який момент хтось може додати коментар, і ніхто не захоче чекати одну годину, щоб побачити його на сайті. В таких випадках, використовуйте стратегію *HTTP-валідації*.

Активація ядра HTTP-кешу Symfony
-------------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Щоб перевірити стратегію HTTP-кешу, увімкніть зворотний зворотний HTTP-проксі Symfony, але тільки в середовищі "розробки" (для "продакшн" середовища ми будемо використовувати "надійніше" рішення):

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -23,3 +23,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Крім того, що це повноцінний зворотний HTTP-проксі, HTTP-проксі Symfony (за допомогою класу ``HttpCache``) додає деяку корисну інформацію налагодження у якості HTTP-заголовків. Це дуже допомагає при валідації встановлених нами заголовків кешу.

Перевірте це на головній сторінці:

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

Для найпершого запиту кеш-сервер повідомляє вам, що відбувся ``miss`` (промах) і що він виконав ``store`` (збереження), щоб закешувати відповідь. Перевірте заголовок ``cache-control``, щоб побачити налаштовану стратегію кешу.

Для наступних запитів відповідь закешовано (``період`` (``age``) також оновлено):

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

Уникнення SQL-запитів за допомогою ESI
-----------------------------------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Слухач ``TwigEventSubscriber`` впроваджує глобальну змінну в Twig для всіх об'єктів конференції. Це відбувається для кожної окремої сторінки веб-сайту. Мабуть, це прекрасне місце для оптимізації.

Ви не будете додавати нові конференції щодня, тому код запитує одні й ті ж дані з бази даних знову і знову.

Ми, можливо, захочемо закешувати імена конференцій і "slugs" за допомогою кешу Symfony, але всякий раз, коли це можливо, я вважаю за краще покладатися на інфраструктуру HTTP-кешування.

Коли ви хочете закешувати фрагмент сторінки, перемістіть його за межі поточного HTTP-запиту, створивши *підзапит*. *ESI* ідеально підходить для цього. ESI — це спосіб вбудувати результат одного HTTP-запиту в інший.

Створіть контролер, який повертає лише фрагмент HTML, що відображає конференції:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -33,6 +33,14 @@ class ConferenceController extends AbstractController
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

Створіть відповідний шаблон:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Перейдіть за посиланням ``/conference_header``, щоб перевірити, що все працює правильно.

.. index::
    single: Twig;render
    single: Twig;path

Час розкрити хитрість! Оновіть макет Twig, щоб викликати щойно створений контролер:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,11 +16,7 @@
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

І *вуаля*! Оновіть сторінку, і веб-сайт все одно відображатиметься так само.

.. tip::

    Використовуйте панель профілювальника Symfony "Request / Response", щоб дізнатися більше про основний запит і його підзапити.

Тепер, щоразу, коли ви переходите на сторінку в браузері, виконуються два HTTP-запити: один для заголовка й один для головної сторінки. Ви погіршили продуктивність. Вітаю!

HTTP-виклик заголовка конференції на даний момент виконується всередині Symfony, тому HTTP-транзакція не відбувається. Це також означає, що немає можливості скористатися заголовками HTTP-кешу.

Перетворіть виклик на "реальний" HTTP за допомогою ESI.

По-перше, увімкніть підтримку ESI:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -13,7 +13,7 @@ framework:
             cookie_samesite: lax
             storage_factory_id: session.storage.factory.native

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Потім використовуйте ``render_esi`` замість ``render``:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,7 +16,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Якщо Symfony виявить зворотний проксі, який знає, як поводитися з ESI, він вмикає підтримку автоматично (якщо ні, то він переходить до синхронної обробки підзапиту).

Оскільки зворотний проксі Symfony підтримує ESI, перевірмо його журнали (спочатку видаліть кеш — див. "Очищення" нижче):

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

Оновіть кілька разів: відповідь за посиланням ``/`` закешовано, а за посиланням ``/conference_header`` — ні. Ми досягли чогось значного: маючи всю сторінку в кеші, одна частина все ще залишається динамічною.

Але це зовсім не те, чого ми хочемо. Закешуйте хедер сторінки на одну годину, незалежно від усього іншого:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -38,7 +38,7 @@ class ConferenceController extends AbstractController
         {
             return $this->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Кеш тепер увімкнено для обох запитів:

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

Заголовок ``x-symfony-cache`` містить два елементи: основний запит ``/`` і підзапит (ESI ``conference_header``). Обидва знаходяться в кеші (``fresh``).

Стратегія кешу головної сторінки та її ESI можуть бути різними. Якщо у нас є сторінка "about", ми можемо зберегти її в кеш на тиждень, і при цьому заголовок буде оновлюватися щогодини.

Видаліть слухача, оскільки він нам більше не потрібен:

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Очищення HTTP-кешу для тестування
----------------------------------------------------------

Тестування веб-сайту в браузері або за допомогою автоматичних тестів стає трохи складнішим з шаром кешування.

Ви можете вручну видалити весь HTTP-кеш, видаливши каталог ``var/cache/dev/http_cache/``:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

Ця стратегія погано працює, якщо ви хочете інвалідувати лише деякі URL-адреси або якщо ви хочете інтегрувати інвалідацію кешу у свої функціональні тести. Додаймо невелику, доступну лише для адміністратора, кінцеву точку HTTP, щоб інвалідувати деякі URL-адреси:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/security.yaml
    +++ b/config/packages/security.yaml
    @@ -17,6 +17,8 @@ security:
                 lazy: true
                 provider: app_user_provider
                 custom_authenticator: App\Security\AppAuthenticator
    +            http_basic: { realm: Admin Area }
    +            entry_point: App\Security\AppAuthenticator
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
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

Новий контролер був обмежений лише HTTP-методом ``PURGE``. Цей метод не входить у стандарт HTTP, але він широко використовується для інвалідації кешів.

За замовчуванням параметри маршруту не можуть містити символ ``/``, оскільки він розділяє сегменти URL-адреси. Ви можете перевизначити це обмеження для останнього параметра маршруту, наприклад ``uri``, встановивши ваш власний шаблон вимог (``.*``).

Те, як ми отримуємо екземпляр ``HttpCache``, також може виглядати дещо дивним; ми використовуємо анонімний клас, оскільки отримати доступ до "реального" неможливо. Екземпляр ``HttpCache`` обертає справжнє ядро, яке не знає про шар кешу, як це і має бути.

Інвалідуйте головну сторінку та хедер конференції за допомогою наступних викликів cURL:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

Підкоманда ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` повертає поточну URL-адресу локального веб-сервера.

.. note::

    Контролер не має імені маршруту, оскільки він ніколи не буде згадуватися в коді.

Групування подібних маршрутів за префіксом
--------------------------------------------------------------------------------

.. index::
    single: Attributes;Route

Два маршрути в адміністративному контролері мають той самий префікс ``/admin``. Замість того щоб повторювати його у всіх маршрутах, виконайте рефакторинг маршрутів, щоб налаштувати префікс у самому класі:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Annotation\Route;
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

Кешування інтенсивних операцій ЦП/пам'яті
-----------------------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

У нас на веб-сайті немає алгоритмів, що інтенсивно використовують ЦП або пам'ять. Щоб поговорити про *локальні кеші*, створімо команду, яка відображає поточний крок, над яким ми працюємо (точніше, ім'я тегу Git, закріпленого за поточною фіксацією Git).

Компонент Symfony Process дозволяє вам виконати команду й отримати результат (стандартний вивід і вивід помилок).

Реалізуйте команду:

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

    Ви могли б використовувати ``make:command``, щоб створити команду:

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Що робити, якщо ми хочемо закешувати вивід на декілька хвилин? Використовуйте кеш Symfony.

І оберніть код логікою кешу:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
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

Процес тепер викликається лише в тому випадку, якщо елемент ``app.current_step`` відсутній у кеші.

Профілювання та порівняння продуктивності
-------------------------------------------------------------------------------

Ніколи не додавайте кеш наосліп. Майте на увазі, що додавання деякого кешу додає рівень складності. Складно передбачити, що буде працювати швидко, а що повільно, ви можете опинитися в ситуації, коли кеш робить ваш застосунок повільнішим.

Завжди вимірюйте вплив додавання кешу за допомогою інструменту профілювання, такого як `Blackfire`_.

Зверніться до кроку про "Продуктивність", щоб дізнатися більше про те, як ви можете використовувати Blackfire для тестування коду перед розгортанням.

Налаштування зворотного проксі-кешу в продакшн
---------------------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Platform.sh;Varnish
    single: Varnish

Замість використання зворотного проксі Symfony у середовищі розробки ми збираємося використовувати "надійніший" зворотний проксі Varnish.

Додайте Varnish у сервіси Platform.sh:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -2,3 +2,12 @@
     database:
         type: postgresql:14
         disk: 1024
    +
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

Використовуйте Varnish у якості основної точки входу в маршрути:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/routes.yaml
    +++ b/.platform/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Нарешті, створіть файл ``config.vcl``, щоб налаштувати Varnish:

.. code-block:: vcl
    :caption: .platform/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Увімкнення підтримки ESI у Varnish
------------------------------------------------------

Підтримка ESI у Varnish має бути увімкнена явно для кожного запиту. Щоб охопити всі запити, Symfony використовує стандартні заголовки ``Surrogate-Capability`` та ``Surrogate-Control`` для узгодження підтримки ESI:

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

Очищення кешу Varnish
---------------------------------

Інвалідація кешу у продакшн, ймовірно, ніколи не знадобиться, за винятком особливих потреб і, можливо, у не-``master`` гілках. Якщо вам потрібно часто очищати кеш, це, ймовірно, означає, що стратегія кешу має бути змінена (шляхом зниження TTL або використовуючи стратегію валідації замість стратегії закінчення терміну дії).

У будь-якому випадку, подивімося, як налаштувати Varnish для інвалідації кешу:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/config.vcl
    +++ b/.platform/config.vcl
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

У реальному житті ви, ймовірно, обмежите IP-адреси, як описано в `документації по Varnish`_.

Тепер очистьте кеш для деяких URL-адрес:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

URL-адреси виглядають дещо дивно через те, що адреси, повернені ``env:url``, вже закінчуються символом ``/``.

.. sidebar:: Йдемо далі

    * `Cloudflare`_, глобальна хмарна платформа;

    * `Документація по HTTP-кешу Varnish`_;

    * `Специфікація ESI`_ та `ресурси ESI для розробників`_;

    * `Модель валідації HTTP-кешу`_;

    * `HTTP-Кеш у Platform.sh`_.

.. _`Blackfire`: https://blackfire.io/
.. _`документації по Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Документація по HTTP-кешу Varnish`: https://varnish-cache.org/docs/index.html
.. _`Специфікація ESI`: https://www.w3.org/TR/esi-lang
.. _`ресурси ESI для розробників`: https://www.akamai.com/us/en/support/esi.jsp
.. _`Модель валідації HTTP-кешу`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP-Кеш у Platform.sh`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
