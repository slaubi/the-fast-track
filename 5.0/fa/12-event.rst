گوش‌دادن به رویدادها
=======================================

چیدمان فعلی، فاقد یک سربرگ پیمایش (navigation header)، جهت بازگشت به صفحه‌ی اصلی یا رفتن از یک کنفرانس به کنفرانس دیگر است.

افزودن یک سربرگ به وب‌سایت
-------------------------------------------------

.. index::
    single: Twig;for
    single: Twig;path

هر چیزی که بخواهد در تمام صفحات وب‌سایت نمایش داده شود، مانند یک سربرگ، باید بخشی از چیدمان پایه‌ی اصلی وب‌سایت باشد:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -6,6 +6,15 @@
             {% block stylesheets %}{% endblock %}
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
             {% block javascripts %}{% endblock %}
         </body>

افزودن این کد به چیدمان به معنی آن است که تمام قالب‌هایی که آن را بسط می‌دهند، باید یک متغیر ``conferences`` تعریف کنند که در کنترلرشان ایجاد شده و به قالب داده می‌شود.

As we only have two controllers, you might do the following:

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -32,9 +32,10 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             return new Response($this->twig->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
             ]));

تصور کنید تعداد زیادی کنترلر دارید و می‌خواهید همین کار را در تک تک آن‌ها تکرار کنید. این کار زیاد عملی نیست و باید راه بهتری وجود داشته باشد.

Twig از مفهوم متغیر‌های جهانی برخوردار است. یک *متغیر جهانی (global variable)* در تمام قالب‌های renderشده در دسترس است. شما می‌توانید این نوع متغیرها را در فایل پیکربندی تعریف کنید، اما این روش تنها برای مقادیر ایستا کارا است. ما برای افزودن تمام کنفرانس‌ها به عنوان متغیر جهانی Twig، یک شنونده (listener) ایجاد خواهیم کرد.

کشف رویدادهای سیمفونی
----------------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

سیمفونی به صورت توکار، دارای یک کامپوننت اعزام‌کننده‌ی رویداد (Event Dispatcher) است. یک اعزام‌کننده، *رویداد‌های* معینی را در زمانی مشخص *اعزام می‌کند* که شنونده‌ها می‌توانند به آن گوش دهند. شنونده‌ها قلاب‌هایی (hooks) به بخش درونی چارچوب هستند.

برای نمونه، برخی از رویداد‌ها به شما اجازه می‌دهند تا با چرخه‌حیات درخواست‌های HTTP، تعامل داشته باشید. در طول رسیدگی به درخواست، اعزام‌کننده زمانی که درخواست ایجاد می‌شود، زمانی که کنترلر می‌خواهد اجرا شود، زمانی که پاسخ برای ارسال آماده است و یا زمانی که یک استثناء پرتاب شده  است،  یک رویداد اعزام می‌کند. یک *شنونده* می‌تواند به یک یا چند رویداد گوش کرده و بر اساس رویداد زمینه، منطقی را اجرا کند.

رویداد‌ها افزونه‌هایی خوش‌تعریف هستند که چارچوب را عمومی‌تر و بسط‌پذیرتر می‌کنند. بسیاری از کامپوننت‌های سیمفونی مثل Security، Messenger، Workflow یا Mailer به صورت گسترده از آن‌ها استفاده می‌کنند.

یکی دیگر از مثال‌های توکار رویدادها و شنونده‌ها در عمل، چرخه‌حیات فرمان (command) است: شما می‌توانید یک شنونده ایجاد کنید که قبل از اجرای هر فرمان، کدی را به اجرا درآورد.

هر بسته یا باندل نیز می‌تواند رویدادهای خود را اعزام کند تا کدش را بسط‌پذیر نماید.

برای جلوگیری از داشتن یک فایل که توصیف‌کننده‌ی این باشد که یک شنونده می‌خواهد به چه رویداد‌هایی گوش کند، یک *مشترک (subscriber)* ایجاد کنید. مشترک، شنونده‌ای است که دارای یک متد استاتیک با نام ``getSubscribedEvents()`` است که پیکربندی این شنونده را باز می‌گرداند. این باعث می‌شود تا مشترک‌ها بتوانند به صورت خودکار در اعزام‌کننده‌ی سیمفونی ثبت شوند.

پیاده‌سازی یک مشترک (Subscriber)
--------------------------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

حالا دیگر روش کار را با تمام وجود درک کرده‌اید، از باندل maker استفاده کنید تا یک مشترک تولید کنید:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

فرمان از شما می‌پرسد که می‌خواهید به چه رویدادهایی را گوش کنید. رویداد ``Symfony\Component\HttpKernel\Event\ControllerEvent`` را انتخاب کنید که دقیقاً قبل از اجرای کنترلر فراخوانی می‌شود. این بهترین زمان برای تزریق متغیر جهانی ``conferences`` است تا هنگامی که کنترلر قالب را render می‌کند، Twig به آن دسترسی داشته باشد. مشترک خود را به صورت زیر به‌روزرسانی کنید:

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
         public function onControllerEvent(ControllerEvent $event)
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }

         public static function getSubscribedEvents()

حالا می‌توانید به هر تعداد که می‌خواهید کنترلر اضافه کنید: متغیر ``conferences`` همواره برای Twig در دسترس خواهد بود.

.. note::

    در گام بعدی در مورد یک روش جایگزین با کارایی بسیار بهتر صحبت خواهیم کرد.

مرتب‌سازی کنفرانس‌ها بر اساس سال و شهر
------------------------------------------------------------------------

مرتب‌کردن کنفرانس‌ها بر اساس سال می‌تواند مرورکردن را تسهیل کند. ما می‌توانیم یک متد سفارشی برای دریافت و مرتب‌سازی تمام کنفرانس‌ها ایجاد کنیم، اما به جای آن می‌خواهیم پیاده‌سازی پیشفرض متد ``findAll()`` را تغییر دهیم تا ماطمینان یابیم که مرتب‌سازی به همه‌جا اعمال می‌گردد:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/ConferenceRepository.php
    +++ b/src/Repository/ConferenceRepository.php
    @@ -19,6 +19,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll()
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         // /**
         //  * @return Conference[] Returns an array of Conference objects
         //  */

در انتهای این گام، وب‌سایت باید مشابه شکل زیر باشد:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: بیشتر بدانید

    * `جریان درخواست-پاسخ <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ در اپلیکیشن‌های سیمفونی؛

    * `رویداد‌های توکار HTTP در سیمفونی <https://symfony.com/doc/current/reference/events.html>`_؛

    * `رویداد‌های توکار کنسول سیمفونی <https://symfony.com/doc/current/components/console/events.html>`_.
