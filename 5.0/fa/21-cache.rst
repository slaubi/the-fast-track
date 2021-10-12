نهان‌سازی به منظور افزایش کارایی
=============================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

افزایش محبوبیت ممکن است همراه با مشکلات کارایی و سرعت باشد. چند نمونه‌ی معمول آن، شاخص‌های گمشده‌ی پایگاه‌داده یا هزاران درخواست SQL برای هر صفحه است. شما با یک پایگاه‌داده‌ی خالی مشکلی نخواهید داشت اما با افزایش ترافیک و رشد داده‌ها، ممکن است بالاخره به  نقطه‌ای برسید که دچار مشکل شوید.

افزودن HTTP Cache Headers
-------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

بهره‌گیری از راهبردهای نهان‌سازی HTTP، یک راه فوق‌العاده برای حداکثرسازی کارایی برای کاربران نهایی است که تلاش کمی می‌طلبد. برای فعال‌سازی نهان‌سازی، یک پروکسی نهان‌ساز معکوس (reverse proxy cache) به محیط عمل‌آوری اضافه کنید و حتی برای کارایی بیشتر، از یک `CDN`_ برای نهان‌سازی در لبه بهره بگیرید.

بیاید صفحه‌ی اصلی را برای یک ساعت نهان‌سازی کنیم:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -35,9 +35,12 @@ class ConferenceController extends AbstractController
          */
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/index.html.twig', [
    +        $response = new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         /**

متد ``setSharedMaxAge()``، انقضای نهان‌سازی را برای پروکسی‌های معکوس پیکربندی می‌کند. برای کنترل نهان‌سازی مرورگر از ``setMaxAge()`` استفاده کنید. زمان بر حسب ثانیه بیان شده است (ا ساعت = ۶۰ دقیقه = ۳۶۰۰ ثانیه).

از آنجایی که صفحه‌ی کنفرانس پویاتر است، نهان‌سازی آن چالش‌برانگیزتر است. هر کسی می‌تواند در هر لحظه‌ای یک کامنت اضافه کند و هیچ‌کس هم نمی‌خواهد که یک ساعت صبر کند تا کامنتش را برخط ببینید. در چنین مواردی از راهبرد *اعتبارسنجی HTTP* استفاده کنید.

فعال‌سازی Symfony HTTP Cache Kernel
---------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

برای آزمودن راهبرد نهان‌سازی HTTP، از پروکسی معکوس HTTP در سیمفونی استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -1,6 +1,7 @@
     <?php

     use App\Kernel;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;

    @@ -21,6 +22,11 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? false) {
     }

     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
    +
    +if ('dev' === $kernel->getEnvironment()) {
    +    $kernel = new HttpCache($kernel);
    +}
    +
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);

پروکسی معکوس HTTP در سیمفونی، در کنار اینکه یک پروکسی معکوس HTTP تمام‌عیار است، مقدار اطلاعات مفید اشکال‌زدایی همچون سربرگ‌های HTTP را نیز اضافه می‌کند (از طریق ``HttpCache``). این موضوع در اعتبارسنجی سربرگ‌هایی که تنظیم کرده‌ایم، بسیار کمک می‌کند.

این را در صفحه‌ی اصلی بررسی کنید:

.. code-block:: bash
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

برای اولین درخواست، سرور نهان‌ساز به شما می‌گوید که یک ``miss`` اتفاق افتاده و یک ``store`` انجام داده تا پاسخ را نهان‌سازی کند. سربرگ ``cache-control`` را بررسی کنید تا ببینید که راهبرد نهان‌سازی پیکربندی شده است.

برای درخواست‌های بعدی، پاسخ نهان‌سازی شده است (همچنین ``age`` به‌روزرسانی شده است):

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

اجتناب از درخواست‌های SQL با استفاده از ESI
--------------------------------------------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

شنونده‌ی ``TwigEventSubscriber``، یک متغیر جهانی را به همراه تمام شیءهای کنفرانس به Twig تزریق می‌کند. این شنونده، اینکار را برای هر یک از صفحات وب‌سایت انجام می‌دهد. این احتمالاً یک هدف عالی برای بهینه‌سازی است.

شما هر روز یک کنفرانس جدید اضافه نمی‌کنید، بنابراین کد بارها و بارها در حال پروس‌وجوی داده‌های یکسان از پایگاه‌داده است.

ما ممکن است که بخواهیم نام و slugهای کنفرانس را به کمک نهان‌ساز سیمفونی، نهان‌سازی کنیم. اما من دوست دارم تا جایی که امکان دارد، به زیرساخت نهان‌سازی HTTP تکیه کنم.

وقتی که می‌خواهید قسمتی از یک صفحه را نهان‌سازی کنید، آن بخش را با ایجاد یک *زیردرخواست* به خارج از درخواست HTTP فعلی انتقال دهید. *ESI* در این وضعیت یک گزینه‌ی کاملاًمنطبق است. ESI راهی برای توکار کردن نتیجه‌ی یک درخواست HTTP در درون یک درخواست دیگر است.

یک کنترلر ایجاد کنید که تنها بخش HTMLای که کنفرانس‌ها را نمایش می‌دهد، برگرداند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -45,6 +45,16 @@ class ConferenceController extends AbstractController
             return $response;
         }

    +    /**
    +     * @Route("/conference_header", name="conference_header")
    +     */
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return new Response($this->twig->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
    +    }
    +
         /**
          * @Route("/conference/{slug}", name="conference")
          */

قالب متناظر را ایجاد کنید:

.. code-block:: twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

به ``/conference_header`` بروید تا بررسی کنید که همه‌چیز درست کار می‌کند.

.. index::
    single: Twig;render
    single: Twig;path

زمان آن است که حقه را آشکار کنیم! قالب Twig را به‌روزرسانی کنید تا کنترلری که همین الان ایجاد کردم را فراخوانی کند:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,11 +8,7 @@
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

بفرمایید، آماده است. صفحه را تازه‌سازی کنید و وب‌سایت هنوز همان چیز‌های قبلی را نشان می‌دهد.

.. tip::

    از پنل «Request / Response» موجود در نمایه‌ساز سیمفونی برای یادگیری بیشتر در رابطه با درخواست اصلی و زیردرخواست‌هایش استفاده کنید.

حالا هر وقت که صفحه‌ای را در مرورگر باز کنید، دو درخواست HTTP اجرا می‌شود، یکی برای سربرگ و دیگری برای صفحه‌ی اصلی. تبریک می‌گوییم، شما کارایی را بدتر کرده‌اید!

در حال حاضر فراخوانی HTTP برای سربرگ کنفرانس به صورت داخلی توسط سیمفونی انجام می‌شود، بنابراین رفت‌وبرگشت HTTP در کار نیست. این همچنین به این معناست که هیچ راهی برای سودبردن از سربرگ‌های نهان‌سازی HTTP وجود ندارد.

با استفاده از ESI، فراخوانی را به یک HTTP «واقعی» تبدیل کنید.

ابتدا پشتیبانی از ESI را فعال کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -11,7 +11,7 @@ framework:
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

سپس به جای ``render`` از ``render_esi`` استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,7 +8,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

اگر سیمفونی یک پروکسی معکوس که قادر به کار با ESIها است را شناسایی کند، به صورت خودکار پشتیبانی را فعال می‌کند (در غیر اینصورت، به renderکردن زیردرخواست به صورت همزمان رو می‌آورد).

از آنجایی که پروکسی معکوس سیمفونی از ESIها پشتیبانی می‌کند، بیایید لاگ‌های آن را بررسی کنیم (ابتدا نهانگاه را پاک کنید - در زیر، «Purging» را ببینید):

.. code-block:: bash
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

چندبار تازه‌سازی کنید: پاسخ ``/`` نهان‌سازی شده است اما ``/conference_header`` نهان‌سازی نشده است. ما به چیزی بزرگ دست‌یافتیم: ذخیره‌ی تمام صفحه در نهانگاه به همراه پویا نگه‌داشتن یک بخش خاص از صفحه.

البته این چیزی نبود که ما می‌خواستیم. مستقل از هر چیز دیگری، صفحه‌ی سربرگ را برای یک ساعت نهان‌سازی کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -48,9 +48,12 @@ class ConferenceController extends AbstractController
          */
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/header.html.twig', [
    +        $response = new Response($this->twig->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         /**

حالا نهان‌سازی برای هر دو درخواست فعال شده است:

.. code-block:: bash
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

سربرگ ``x-symfony-cache`` شامل دو جزء است: درخواست اصلی ``/`` و یک زیردرخواست (ESI مربوط به ``conference_header``). هر در نهانگاه هستند (``fresh``).

راهبرد نهان‌سازی می‌تواند با صفحه‌ی اصلی و ESIهایش متفاوت باشد. اگر یک صفحه‌ی «درباره‌ی ما» داشته باشیم، ممکن است بخواهیم که آن را تا یک هفته در نهانگاه ذخیره کنیم و همچنین بخواهیم که سربرگ صفحه هر ساعت بهٰ‌روز شود.

شنونده را حذف کنید چرا که دیگر به آن نیازی نداریم:

.. code-block:: bash

    $ rm src/EventSubscriber/TwigEventSubscriber.php

پاکسازی نهنگاه HTTP به منظور آزمودن وب‌سایت
-----------------------------------------------------------------------------

با وجود لایه‌ی نهان‌سازی، آزمودن وب‌سایت در یک مرورگر یا از طریق آزمون‌های خودکار، کمی سخت‌تر است.

You can manually remove all the HTTP cache by removing the
``var/cache/dev/http_cache/`` directory:

.. code-block:: bash

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Annotations;@Route

اگر بخواهید تنها بخشی از URLها را باطل کنید یا بخواهید باطل‌کردن نهانگاه را با آزمون‌های کارکردی خود ادغام کنید، این راهبرد خوب کار نمی‌کند.بیایید یک پایانه کوچک HTTP برای باطل‌کردن بخشی از URLها اضافه کنیم که تنها در دسترس مدیر قرار دارد.

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -6,8 +6,10 @@ use App\Entity\Comment;
     use App\Message\CommentMessage;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
    @@ -54,4 +56,19 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]);
         }
    +
    +    /**
    +     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     */
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store = (new class($kernel) extends HttpCache {})->getStore();
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

این کنترلر جدید، به متدِ HTTP از نوع ``PURGE`` محدود شده است. این نوع متد در استاندارد HTTP وجود ندارد اما به صورت گسترده برای باطل‌کردن نهان‌سازی مورد استفاده قرار می‌گیرد.

به صورت پیشفرض، پارامترهای راه نمی‌توانند شامل ``/`` باشند چرا که این کاراکتر بخش‌های URL را از هم جداسازی می‌کند. می‌توانید این محدودیت را برای آخرین پارامتر راه، مثل ``uri``، با تنظیم‌کردن الگوی مورد نیاز خود باطل کنید (``.*``).

راهی که ما از طریق آن نمونه‌ی ``HttpCache`` را می‌گیریم، کمی عجیب به نظر می‌رسد؛ما از یک کلاس ناشناس استفاده می‌کنیم زیرا دسترسی به کلاس واقعی امکان‌پذیر نیست. نمونه‌ی ``HttpCache`` حاوی هسته‌ی واقعی است که به درستی از لایه‌ی نهان‌سازی ناآگاه است.

صفحه‌ی اصلی و سربرگ کنفرانس را به کمک فراخوانی‌های cURL زیر باطل کنید:

.. code-block:: bash

    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

The ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` sub-command returns the current URL of the local web server.

.. note::

    کنترلر نامِ راه (route name) ندارد، چرا که هرگز در کد به آن ارجاع نخواهد شد.

گروه‌بندی راه‌های مشابه با یک پیشوند
---------------------------------------------------------------------

.. index::
    single: Annotations;@Route

دو راهی که در کنترلر مدیر قرار دارند، دارای پیشوند یکسان ``/admin`` هستند.  به جای تکرار این پیشوند در تمام راه‌ها، راه‌ها را refactor کنید تا پیشوند را بر روی خود کلاس پیکربندی کند:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,9 @@ use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    +/**
    + * @Route("/admin")
    + */
     class AdminController extends AbstractController
     {
         private $twig;
    @@ -29,7 +32,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/comment/review/{id}", name="review_comment")
    +     * @Route("/comment/review/{id}", name="review_comment")
          */
         public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
         {
    @@ -58,7 +61,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     * @Route("/http-cache/{uri<.*>}", methods={"PURGE"})
          */
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
         {

نهان‌سازی عملیات‌های پرمصرف CPU/Memory
-----------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

ما بر روی این وب‌سایت الگوریتم‌هایی نداریم که CPU یا حافظه‌ی زیادی مصرف کنند. بیایید برای صحبت درباره‌ی *نهان‌سازی محلی*، یک فرمان ایجاد کنیم که گام فعلی‌ای که در حال کار بر روی آن هستیم (اگر بخواهیم دقیق‌تر بگوییم، نام تگ Git که به commit فعلی Git ضمیمه شده است) را نمایش دهد.

کامپوننت سیمفونی Process به ما اجازه می‌دهد تا  یک فرمان را اجرا کنیم و نتیجه را بگیریم (خروجی‌ استاندارد و خطا)؛ این کامپوننت را نصب کنید:

.. code-block:: bash

    $ symfony composer req process

فرمان را پیاده‌سازی کنید:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    class StepInfoCommand extends Command
    {
        protected static $defaultName = 'app:step:info';

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return 0;
        }
    }

.. index::
    single: Command;make:command

.. note::

    می‌توانستید از فرمان ``make:command`` برای ایجاد فرمان استفاده کنید:

    .. code-block:: bash
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

چه می‌شود اگر بخواهیم خروجی را برای چند دقیقه نهان‌سازی کنیم؟ از نهان‌ساز سیمفونی استفاده کنید:

.. code-block:: bash

    $ symfony composer req cache

و کد را با منطق نهان‌سازی در بر بگیرید:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
    @@ -6,16 +6,31 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     class StepInfoCommand extends Command
     {
         protected static $defaultName = 'app:step:info';

    +    private $cache;
    +
    +    public function __construct(CacheInterface $cache)
    +    {
    +        $this->cache = $cache;
    +
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

             return 0;
         }

حالا فرآیند تنها در صورتی فراخوانی می‌شود که ``app.current_step`` در نهانگاه نباشد.

Profiling و مقایسه‌ی کارایی
-------------------------------------------

هرگز به صورت کورکورانه نهان‌سازی نکنید. به یاد داشته باشید که افزودن نهان‌سازی، یک لایه از پیچیدگی را ایجاد می‌کند. و از آنجایی که همه‌ی ما در حدس‌زدن اینکه چه چیزی سریع و چه چیزی کند خواهد بود، بسیار عملکرد بدی داریم، ممکن است در نهایت به وضعیتی برسیم که نهان‌سازی اپلیکیشن ما را کندتر کند.

همواره اثر افزودن یک نهان‌سازی را با یک ابزار نمایه‌ساز همچون `Blackfire <https://blackfire.io/>`_. اندازه‌گیری کنید.

برای یادگیری اینکه چگونه می‌توانید از Blackfire برای آزمودن کدتان قبل از استقرار استفاده کنید، به گام مربوط به «کارایی (Performance)» مراجعه کنید.

پیکربندی پروکسی معکوس نهان‌ساز در محیط عمل‌آوری
------------------------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: SymfonyCloud;Varnish
    single: Varnish

در محیط عمل‌آوری از پروکسی معکوس خود سیمفونی استفاده نکنید. همواره یک پروکسی معکوس همچون Varnish را بر روی زیرساخت خود ترجیح دهید یا از یک CDN تجاری استفاده کنید.

Varnish را به سرویس‌های SymfonyCloud بیافزایید:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -10,3 +10,12 @@ queue:
         type: rabbitmq:3.7
         disk: 1024
         size: S
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
    single: SymfonyCloud;Routes

از Varnish به عنوان مدخل اصلی راه‌ها استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

در پایان برای پیکربندی Varnish، یک فایل ``config.vcl`` ایجاد کنید:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

فعال‌سازی پشتیبانی از ESI در Varnish
----------------------------------------------------------

پشتیبانی از ESI در Varnish باید به صورت صریح برای هر درخواست فعال گردد. برای همگانی‌کردن آن، سیمفونی از سربرگ‌های استاندارد ``Surrogate-Capability`` و ``Surrogate-Control`` برای برقراری پشتیبانی از ESI استفاده می‌کند:

.. code-block:: vcl
    :caption: .symfony/config.vcl

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

پاکسازی نهانگاه Varnish
-------------------------------------

قاعدتاً باطل‌کردن نهانگاه در محیط عمل‌آوری هرگز نباید مورد نیاز باشد، مگر در شرایط اضطراری و شاید بر روی شاخه‌های non-``master``. اگر نیاز دارید که نهانگاه را بارها پاک کنید، این احتمالاً به این معناست که راهبرد نهان‌سازی باید اصلاح گردد (با کاهش TTL یا با استفاده از یک راهبرد اعتبارسنجی به جای منقضی کردن).

به هر صورت بیایید ببینیم که چگونه می‌توان باطل‌سازی نهانگاه Varnish را پیکربندی کرد:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
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

در شرایط واقعی، احتمالاً شما IPها را همانگونه که در `مستندات Varnish <https://varnish-cache.org/docs/trunk/users-guide/purging.html>`_ تشریح شده است، محدود می‌کنید.

حالا تعدادی URL را پاکسازی کنید:

.. code-block:: bash

    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`
    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`conference_header

URLها کمی عجیب به نظر می‌رسد زیرا URLهایی که توسط ``env:urls`` بازگردانده می شوند با ``/`` پایان می‌یابند.

.. sidebar:: بیشتر بدانید

    * `Cloudflare <https://www.cloudflare.com>`_، پلتفرم ابری جهانی؛

    * `مستندات نهان‌سازی HTTP در Varnish <https://varnish-cache.org/docs/index.html>`_؛

    * `مشخصات فنی ESI <https://www.w3.org/TR/esi-lang>`_ و `منابع مخصوص توسعه‌دهندگان ESI <https://www.akamai.com/us/en/support/esi.jsp>`_؛

    * `مدل اعتبارسنجی نهان‌سازی HTTP <https://symfony.com/doc/current/http_cache/validation.html>`_؛

    * `HTTP Cache in SymfonyCloud <https://symfony.com/doc/current/cloud/cookbooks/cache.html>`_.

.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
