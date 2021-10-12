إنشاء وحدة تحكم
============================

.. index::
    single: Controller
    single: Routing;Route

إن مشروع دفتر الضيوف الخاص بنا موجود بالفعل علي خوادم الانتاجية ولكننا قمنا بالخداع قليلاً. لا يحتوي الموقع علي أي صفحات بعد. الصفحة الرئيسية عبارة عن صفحة خطأ 404. هيا نقوم بتصليح ذلك.

عندما يأتي طلب HTTP، مثل الصفحة الرئيسية (``http://localhost:8000/``)، يحاول سيمفوني العثور علي *مسار* يطابق *مسار الطلب* (``/``). *المسار* هو رابط بين مسار الطلب ودالة *PHP* التي تقوم بعمل *الرد* علي طلب ال HTTP.

تسمي هذه الدوال القابلة للاستدعاء "بوحدات التحكم". في سيمفوني، أغلب وحدات التحكم يتم كتابتها كفئات PHP. يمكنك إنشاء هذه الفئة يدويا، ولكننا نحب ان نتحرك بسرعة، فدعنا نري كيف يمكن لسيمفوني ان يساعدنا.

كن كسول مع Maker Bundle
-------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

لانشاء وحدات تحكم دون اي عناء، يمكننا استخدام حزمة ``symfony/maker-bundle``:

.. code-block:: bash

    $ symfony composer req maker --dev

بما ان حزمة المُصنع (maker bundle) مفيدة فقط اثناء عملية التطوير، فلا تنسي اضافة عَلم ``--dev`` لتجنب تفعيلها في الانتاجية.

تساعدك حزمة المُصنع علي انشاء العديد من وحدات التحكم المختلفة. سنستخدمها طوال الوقت في هذا الكتاب. كل "مولد" مُعرف في أمر وكل الاوامر جزء من مساحة اسم امر ``make``.

.. index::
    single: Command;list

يقوم امر ``list`` المدمج مع سيمفوني بسرد جميع الاوامر الموجودة في مساحة الاسم المعطاة؛ استخدمه لتقوم باكتشاف جميع المولدات التي توفرها حزمة maker.

.. code-block:: bash
    :class: ignore

    $ symfony console list make

إختيار شكل الإعدادات
--------------------------------------

قبل عمل اول مُتحكم للمشروع، نحتاج الي تحديد تنسيق الاعدادات التي نريد استخدامها. يدعم سيمفوني YAML، XML، PHP و annotations خارج الصندوق.

يعتبر *YAML* افضل اختيار من اجل *الاعدادات الخاصة بالحزم*. هذا هو التنسيق المستخدم في مجلد ``config/``. في الغالب عندما تقوم بتنصيب حزمة جديدة ستقوم وصفة هذه الحزمة بإضافة ملف جديد ينتهي بــ``.yaml`` الي هذا المجلد.

*بالنسبة للاعدادات الخاصة برمز الـ PHP يعتبرافضل اختيار هو *annotations* حيث انه يتم تعريفها بجوار الرمز. اسمحوا لي ان اشرح هذا بمثال. عندما يأتي طلب بعد الاعدادات تحتاج ان تخبر سيمفوني بالمُتحكم (PHP class) الخاص بمعالجة مسار هذا الطلب. عند استخدام YAML، XML، او PHP كتنسيقات الاعدادات يتم اشراك ملفين (ملف الاعدادات وملف مُتحكم PHP). عند استخدام annotations الاعدادات تتم مباشرة في فئة المُتحكم.

لادارة الـannotations نحتاج لاضافة تَبعية أخري:

.. code-block:: bash

    $ symfony composer req annotations

قد تتساءل كيف يمكنك تخمين اسم الحزمة التي تحتاج لتثبيتها من اجل ميزة؟ أغلب الاحيان لا تحتاج أن تعرف. في معظم الحالات تحتوي رسائل الخطأ في سيمفوني علي اسم الحزمة التي تحتاج لثبيتها. علي سبيل المثال تشغيل ``symfony make:controller`` دون وجود حزمة ``annotations`` سوف ينتهي برسالة خطأ تشمل تلميح عن تثبيت الحزمة الصحيحة.

إنتاج وحدة تحكم (مُتحكم)
-------------------------------------------

.. index::
    single: Command;make:controller

إنشاء اول *وحدة تحكم* خاصة بك عن طريق امر ``make:controller``:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;@Route

ينشئ الامر فئة ``ConferenceController`` تحت مجلد الـ ``src/Controller/``. تحتوي الفئة التي تم انشاؤها من بعض الرموز (الكود) النمطية (الصيغة التشكلية) الجاهزة ليتم تحسينها (ضبطها):

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

تعتبر الشرح التوضيحي ``@Route("/conference", name="conference")`` هو ما يجعل دالة الـ ``index()`` وحدة تحكم (مُتحكم) (الاعدادات بجوار الرمز البرمجي التي تقوم بضبطه).

عندما تضغط ``/conference`` في المتصفح يتم تنفيذ وحدة التحكم ويعُاد الرد.

قم بتعديل الرابط ليتطابق مع الصفحة الرئيسية:

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

*اسم* المسار سوف يكون مفيد عندما نريد ان نُشير الي الصفحة الرئيسية في الرمز البرمجي. بدلاً من ترميز مسار الـ ``/`` بصعوبة سوف نستخدم اسم المسار.

بدلاً من الصفحة الإفتراضية المعروضة فلنرجع صفحة HTML بسيطة:

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

تحديث المتصفح:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

المسئولية الرئيسية لوحدة التحكم هي اعادة ``رد`` HTTP للطلب.

.. _easter-egg:

إضافة بيضة عيد الربيع
---------------------------------------

لتوضيح كيف تستفيد الاستجابة من المعلومات الواردة من الطلب سنقوم بإضافة `Easter egg`_. عندما تحتوي الصفحة الرئيسية علي جملة استعلام مثل ``?hello=Fabien``، سنقوم بإضفة نص لتحية الشخص:

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

يكشف سيمفوني بيانات الطلب عن طريق كائن ``Request``. عندما يري سيمفوني دلالة وحدة تحكم بهذا التلميح، يعرف تلقائياً ان يمررها اليك. يمكننا ان نستخدما للحصول علي عنصر الـ ``name`` من جملة الاستعلام واضافة عنوان ``<h1>``.

قم بضغط ``/`` ومن ثم ``/?hello=Fabien`` في المتصفح لتري الفرق.

.. note::

    إنتبه لمناداة ``htmlspecialchars()`` لتجنب مشاكل الـ XSS. هذا شئ سوف يتم تنفيذه بشكل تلقائي عندما نقوم بالتغير الي محرك قالب مناسب.

كان من الممكن ايضا ان نجعل الاسم جزي من ال URL:

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

جزء الـ ``{name}`` الموجود في المسار يعتبر *معامل مسار* ديناميكي - وهو يعمل كبديل. يمكن الان ضغط ``/hello`` ومن ثم ``/hello/Fabien`` في المتصفح للحصول علي النتائج نفسها التي حصلت عليها من قبل. يمكنك الحصول علي *قيمة* معامل الـ ``{name}`` بإضافة دلالة في وحدة التحكم بنفس *الاسم*. إذاً، ``$name``.

.. sidebar:: الذهاب أبعد من ذلك

    * نظام `المسارات <https://symfony.com/doc/current/routing.html>`_ الخاص بسيمفوني؛

    * `SymfonyCasts Routes, Controllers &Pages tutorial <https://symfonycasts.com/screencast/symfony/route-controller>`_؛

    * `الشروحات <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_؛ في PHP؛

    * مكون الـ `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_؛

    * `XSS (Cross-Site Scripting) <https://owasp.org/www-community/attacks/xss/>`_ security attacks؛

    * The `Symfony Routing Cheat Sheet <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`Easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
