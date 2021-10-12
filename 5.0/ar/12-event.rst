الاستماع إلى الأحداث
======================================

يفتقد التخطيط الحالي إلى رأس تنقل للعودة إلى الصفحة الرئيسية أو التبديل من مؤتمر إلى آخر.

إضافة رأس موقع header
---------------------------------

.. index::
    single: Twig;for
    single: Twig;path

يجب أن يكون أي شيء يجب عرضه على جميع صفحات الويب ، مثل الرأس ، جزءًا من التخطيط الأساسي الرئيسي:

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

إن إضافة هذا الكود إلى التخطيط يعني أن جميع القوالب التي توسعه يجب أن تحدد متغير ``المؤتمرات``، الذي يجب إنشاؤه وتمريره من وحدات التحكم الخاصة بهم.

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

تخيل أنك مضطر لتحديث العشرات من وحدات التحكم. وتفعل نفس الشيء مع كل ما هو جديد. هذه ليست عملية للغاية. يجب أن تكون هناك طريقة أفضل.

لدى Twig مفهوم المتغيرات العالمية.  * متغير عام * متاح في جميع النماذج المقدمة. يمكنك تحديدها في ملف تكوين ، ولكنها تعمل فقط للقيم الثابتة. لإضافة جميع المؤتمرات كمتغير عالمي Twig ، سنقوم بإنشاء مستمع.

اكتشاف أحداث سيمفوني Symfony Events
-----------------------------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

يأتي سيمفوني مدمجًا مع مكون مرسل الأحداث (Event Dispatcher Component). يقوم المرسل * بإرسال * أحداث معينة * في أوقات محددة يمكن * للمستمعين * الاستماع إليها. المستمعون هم خطافات في الإطار الداخلي.

على سبيل المثال ، تسمح لك بعض الأحداث بالتفاعل مع دورة حياة طلبات HTTP. أثناء معالجة الطلب ، يرسل المرسل الأحداث عند إنشاء الطلب ، أو عندما تكون وحدة التحكم على وشك التنفيذ ، أو عندما تكون الاستجابة جاهزة للإرسال ، أو عندما يتم طرح استثناء. يمكن * المستمع * الاستماع إلى حدث واحد أو أكثر وتنفيذ بعض المنطق بناءً على سياق الحدث.

الأحداث عبارة عن نقاط امتداد محددة جيدًا تجعل الإطار أكثر شمولاً وقابلية للتوسيع. تستخدم العديد من مكونات سيمفوني مثل Security أو Messenger أو Workflow أو Mailer على نطاق واسع.

مثال آخر مضمن للأحداث والمستمعين قيد التنفيذ هو دورة حياة الأمر: يمكنك إنشاء مستمع لتنفيذ التعليمات البرمجية قبل تشغيل *أي أمر*.

يمكن لأي حزمة أو حزمة أيضًا إرسال أحداثها الخاصة لجعل رمزها قابلاً للتوسيع.

لتجنب وجود ملف تكوين يصف الأحداث التي يرغب المستمع في الاستماع إليها ، قم بإنشاء *مشترك*. المشترك هو مستمع بأسلوب ثابت `` getSubscribeEvents () `` يعيد تكوينه. هذا يسمح للمشتركين بالتسجيل في مرسل سيمفوني تلقائيًا.

إنجاز مشترك Subscriber
--------------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

أنت تعرف الأغنية عن ظهر قلب الآن ، استخدم حزمة صانع لإنشاء مشترك Subscriber:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

يسألك الأمر عن الحدث الذي تريد الاستماع إليه. اختر حدث `` Symfony \ Component \ HttpKernel \ Event \ ControllerEvent `` ، الذي يتم إرساله قبل استدعاء وحدة التحكم. هذا هو أفضل وقت لحقن المتغير العالمي `` للمؤتمرات '' حتى يتمكن Twig من الوصول إليه عندما تقوم وحدة التحكم بتقديم القالب. قم بتحديث المشترك كما يلي:

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

الآن ، يمكنك إضافة أي عدد تريده من وحدات التحكم: سيكون متغير متغيرات `` المؤتمرات `` متاحًا دائمًا في Twig.

.. note::

    سوف نتحدث عن أداء بديل أفضل بكثير في خطوة لاحقة.

فرز المؤتمرات حسب السنة والمدينة
------------------------------------------------------------

ترتيب قائمة المؤتمرات حسب السنة قد يسهل التصفح. يمكننا إنشاء طريقة مخصصة لاسترداد جميع المؤتمرات وفرزها ، ولكن بدلاً من ذلك ، سنقوم بإلغاء التطبيق الافتراضي لطريقة `` findAll () `` للتأكد من تطبيق الفرز في كل مكان:

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

في نهاية هذه الخطوة ، يجب أن يبدو موقع الويب كما يلي:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: الذهاب أبعد من ذلك

    * `تدفق الطلب-الاستجابة <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ في تطبيقات Symfony؛

    * `أحداث Symfony HTTP المضمنة <https://symfony.com/doc/current/reference/events.html>`_؛

    * أحداث `Symfony Console المدمجة <https://symfony.com/doc/current/components/console/events.html>`_.
