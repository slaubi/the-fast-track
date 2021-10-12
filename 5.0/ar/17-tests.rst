الإختبار
================

.. index::
    single: PHPUnit

حيث اننا نقوم بإضافه المزيد من الوظائف للتطبيق. يبدو انه الوقت المناسب للتحدث عن الاختبارات (Tests)

*حقيقة مضحكة*: لقد وجدت خطأ أثناء كتابة الاختبارات في هذا الفصل.

سيمفوني يعتمد علي PHPUnit لاختبار الوحدات. لنقم بتنصيبه

.. code-block:: bash

    $ symfony composer req phpunit --dev

كتابة وحدات الاختبار
--------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

``SpamChecker`` هو أول شئ سنقوم بكتابة إختبارات له. لصنع وحدة :

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

اختبار كاشف الزيف تحدي. حيث اننا بالتاكيد لا نريد ان نرسل لـ Akismet API. لذلك سنصنع *mock* للـAPI

.. index::
    single: Mock

لنقوم بكتابة أول اختبار عندما تعطينا الـ API خطأ:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/SpamCheckerTest.php
    +++ b/tests/SpamCheckerTest.php
    @@ -2,12 +2,26 @@

     namespace App\Tests;

    +use App\Entity\Comment;
    +use App\SpamChecker;
     use PHPUnit\Framework\TestCase;
    +use Symfony\Component\HttpClient\MockHttpClient;
    +use Symfony\Component\HttpClient\Response\MockResponse;
    +use Symfony\Contracts\HttpClient\ResponseInterface;

     class SpamCheckerTest extends TestCase
     {
    -    public function testSomething()
    +    public function testSpamScoreWithInvalidRequest()
         {
    -        $this->assertTrue(true);
    +        $comment = new Comment();
    +        $comment->setCreatedAtValue();
    +        $context = [];
    +
    +        $client = new MockHttpClient([new MockResponse('invalid', ['response_headers' => ['x-akismet-debug-help: Invalid key']])]);
    +        $checker = new SpamChecker($client, 'abcde');
    +
    +        $this->expectException(\RuntimeException::class);
    +        $this->expectExceptionMessage('Unable to check for spam: invalid (Invalid key).');
    +        $checker->getSpamScore($comment, $context);
         }
     }

كائن الـ``MockHttpClient`` يمكننا من تقليد اي خادم HTTP. حيث يستقبل مصفوفه من كائنات ``MockResponse`` التي تحتوي علي المحتوي المتوقع و رؤوس الاستجابه (Response headers)

بعد ذلك، نستدعي دالة ``getSpamScore()`` و نتحقق من ظهور الخطآ عن طريق دالة ``expectException()`` من PHPUnit

قم بتشغيل الاختبارات للتحقق من نجاحها:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

لنقم بإضافة اختبارات للمسار السعيد:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/SpamCheckerTest.php
    +++ b/tests/SpamCheckerTest.php
    @@ -24,4 +24,32 @@ class SpamCheckerTest extends TestCase
             $this->expectExceptionMessage('Unable to check for spam: invalid (Invalid key).');
             $checker->getSpamScore($comment, $context);
         }
    +
    +    /**
    +     * @dataProvider getComments
    +     */
    +    public function testSpamScore(int $expectedScore, ResponseInterface $response, Comment $comment, array $context)
    +    {
    +        $client = new MockHttpClient([$response]);
    +        $checker = new SpamChecker($client, 'abcde');
    +
    +        $score = $checker->getSpamScore($comment, $context);
    +        $this->assertSame($expectedScore, $score);
    +    }
    +
    +    public function getComments(): iterable
    +    {
    +        $comment = new Comment();
    +        $comment->setCreatedAtValue();
    +        $context = [];
    +
    +        $response = new MockResponse('', ['response_headers' => ['x-akismet-pro-tip: discard']]);
    +        yield 'blatant_spam' => [2, $response, $comment, $context];
    +
    +        $response = new MockResponse('true');
    +        yield 'spam' => [1, $response, $comment, $context];
    +
    +        $response = new MockResponse('false');
    +        yield 'ham' => [0, $response, $comment, $context];
    +    }
     }

يسمح موفر البيانات (PHPUnit data providers) باستخدام نفس الاختبار لتجربة اكثر من حالة.

لنكتب الاختبار الوظيفي للـمتحكمات (Controllers)
-----------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

اختبار وحدات التحكم يختلف قليلا عن اختبار كائن PHP "عادي" حيث اننا نريد ان نشغل وحدات التحكم في سياق طلب من الخادم (HTTP Request)

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

لنصنع اختبار وظيفي لوحده تحكم المؤتمرات:

.. code-block:: php
    :caption: tests/Controller/ConferenceControllerTest.php

    namespace App\Tests\Controller;

    use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

    class ConferenceControllerTest extends WebTestCase
    {
        public function testIndex()
        {
            $client = static::createClient();
            $client->request('GET', '/');

            $this->assertResponseIsSuccessful();
            $this->assertSelectorTextContains('h2', 'Give your feedback');
        }
    }

الإختبار الأول يتحقق ان الصفحة الرئيسية تعطي استجابه بكود 200.

متغير الـ ```$client``` يحاكي متصفح الانترنت. ف بدلا من ارسال طلبات HTTP للخادم، يقوم بإرسالها الي تطبيق سيمفوني مباشرة. هذه الإستراتيجيه لها فوائد كثيره: إنها اسرع من الطلبات بين الخادم و العميل، و ايضا تسمح باستكشاف حاله الخدمات بعد كل طلب (HTTP Request)

التاكيدات مثل ``assertResponseIsSuccessful`` موجوده فوق PHPUnit لتسهل عليك العمل. يوجد الكثير من التاكيدات المعرفه من سيمفوني

.. tip::

    قمنا باستخدام ``/`` كرابط بدلا من صنعه عن طريق وحدة التوجيه (Router). تم ذلك عن قصد لأن اختبار الروابط للمستخدم النهائي هو جزء مما نريد اختباره. فلو قمت بتغيير مسار الرابط لاحقا. سيفشل الاختبار كتذكير لطيف انه يجب عليك تحويل المستخدم من الرابط القديم للرابط الجديد حتي تكون لطيف مع محركات البحث و المواقع التي تستخدم الرابط القديم لموقعك.

.. note::

    كان يمكننا صنع الاختبار عن طريق حزمة الصانع:

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

اختبارات PHPUnit يتم تشغلها في بيئه `اختبار` منفصله. يجب ان نضع قيمة ``AKISMET_KEY`` السريه لبيئة الاختبار:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    عندما يفشل اختبار، قد يكون من المفيد التحقق من كائن الاستجابه (Response object). يمكنك ان تصل اليه عن طريق ``$client->getResponse()`` و ``echo`` لتري كيف يبدو.

تعريف التركيبات
-----------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

لتتمكن من اختبار قائمة التعليقات، ترقيم الصفحات، و نموذج التعليقات، نحتاج الى ملئ قاعدة البيانات ببعض التعليقات. ونريد أن تكون البيانات مستقرة عند تشغيل الإختبارات كي تنجح عن التشغيل. التركيبات تحديدا هي ما نحتاجه.

لنقوم بتنصيب حزمة تركيبات Doctrine:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

مجلد جديد ``src/DataFixtures/``  تم انشاؤه اثناء تنصيب الحزمه مع كائن بسيط. جاهز للتعديل. لنقوم باضافه مؤتمرين و تعليق واحد الان:

.. code-block:: diff
    :caption: patch_file

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
    @@ -2,6 +2,8 @@

     namespace App\DataFixtures;

    +use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\FixturesBundle\Fixture;
     use Doctrine\Persistence\ObjectManager;

    @@ -9,8 +11,24 @@ class AppFixtures extends Fixture
     {
         public function load(ObjectManager $manager)
         {
    -        // $product = new Product();
    -        // $manager->persist($product);
    +        $amsterdam = new Conference();
    +        $amsterdam->setCity('Amsterdam');
    +        $amsterdam->setYear('2019');
    +        $amsterdam->setIsInternational(true);
    +        $manager->persist($amsterdam);
    +
    +        $paris = new Conference();
    +        $paris->setCity('Paris');
    +        $paris->setYear('2020');
    +        $paris->setIsInternational(false);
    +        $manager->persist($paris);
    +
    +        $comment1 = new Comment();
    +        $comment1->setConference($amsterdam);
    +        $comment1->setAuthor('Fabien');
    +        $comment1->setEmail('fabien@example.com');
    +        $comment1->setText('This was a great conference.');
    +        $manager->persist($comment1);

             $manager->flush();
         }

عندما نقوم بجلب التركيبات، كل البيانات سيتم مسحها; ايضا المستخدم admin. لتجنب ذلك، لنقوم بإضافة مستخدم admin في التركيبات:

.. code-block:: diff

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
    @@ -2,13 +2,22 @@

     namespace App\DataFixtures;

    +use App\Entity\Admin;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\FixturesBundle\Fixture;
     use Doctrine\Persistence\ObjectManager;
    +use Symfony\Component\Security\Core\Encoder\EncoderFactoryInterface;

     class AppFixtures extends Fixture
     {
    +    private $encoderFactory;
    +
    +    public function __construct(EncoderFactoryInterface $encoderFactory)
    +    {
    +        $this->encoderFactory = $encoderFactory;
    +    }
    +
         public function load(ObjectManager $manager)
         {
             $amsterdam = new Conference();
    @@ -30,6 +39,12 @@ class AppFixtures extends Fixture
             $comment1->setText('This was a great conference.');
             $manager->persist($comment1);

    +        $admin = new Admin();
    +        $admin->setRoles(['ROLE_ADMIN']);
    +        $admin->setUsername('admin');
    +        $admin->setPassword($this->encoderFactory->getEncoder(Admin::class)->encodePassword('admin', null));
    +        $manager->persist($admin);
    +
             $manager->flush();
         }
     }

.. index::
    single: Command;debug:autowiring
    single: Debug;Container
    single: Container;Debug

.. tip::

    اذا كنت لا تتذكر أي خدمة تحتاجها لمهمة معينة. استخدم ``debug:autowiring`` مع بعض الكلمات:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

تحميل التركيبات
-----------------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

استخراج البيانات من الموقع في الاختبارات الوظيفية
--------------------------------------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

كما رأينا من قبل، الـ (HTTP Client) المتسخدم في الاختبارات يحاكي متصفح الانترنت. لذلك نستطيع  التنقل في الموقع كاننا نستخدم المتصفح الغير مرئي (headless browser).

لنقوم بإضافة اختبار يضغط علي رابط المؤتمر من الصفحة الرئيسية:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -14,4 +14,19 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }
    +
    +    public function testConferencePage()
    +    {
    +        $client = static::createClient();
    +        $crawler = $client->request('GET', '/');
    +
    +        $this->assertCount(2, $crawler->filter('h4'));
    +
    +        $client->clickLink('View');
    +
    +        $this->assertPageTitleContains('Amsterdam');
    +        $this->assertResponseIsSuccessful();
    +        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    +        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +    }
     }

لنقم بوصف ماذا حدث في الاختبار بشكل بسيط:

* كما الاختبار الأول، قمنا بالذهاب للصفحة الرئيسية؛

* دالة الطلب ``request()`` تعطي كائن زاحف ``Crawler`` يساعدنا في إيجاد عناصر في الصفحة (مثل الروابط، النماذج، اي شئ يمكن الوصول اليه بمحدد CSS او XPath)؛

* بفضل محدد CSS، نقوم بتاكيد ان لدينا مؤتمرين فقط في الصفحة الرئيسية؛

* بعد ذلك نضغط علي عرض الرابط "View" (بما انه لا يمكن الضغط علي اكثر من رابط في وقت واحد. سيمفوني تلقائي يختار أول عنصر من القائمة)؛

* قمنا بالتاكد من عنوان الصفحة، المحتوي  و ``<h2>`` للتاكد من اننا في الصفحة الصحيحة (يمكننا ايضا التاكد من ان رابط الصفحة مطابق للرابط المتوقع)؛

* في النهايه، قمنا بالتاكد انه يوجد تعليق واحد في الصفحه. ``div:contains()`` ليس من المحددات (CSS selector)، ولكن سيمفوني لديها بعض الاضافات، تم استعارتها من jQuery.

بدلا من الضغط علي النص (مثل ``View``)، كان يمكننا تحديد الرابط عن طريق محدد css:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

لنتاكد من أن الاختبار الجديد أخضر:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

لنعمل مع اختبار قاعدة البيانات
--------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env.test

By default, tests are run in the ``test`` Symfony environment as defined in the ``phpunit.xml.dist`` file:

.. code-block:: xml
    :caption: phpunit.xml.dist
    :class: ignore

    <phpunit>
        <php>
            <server name="APP_ENV" value="test" force="true" />
        </php>
    </phpunit>

If you want to use a different database for your tests, override the ``DATABASE_URL`` environment variable in the ``.env.test`` file:

.. code-block:: diff
    :class: ignore

    --- a/.env.test
    +++ b/.env.test
    @@ -1,4 +1,5 @@
     # define your env variables for the test env here
    +DATABASE_URL=postgres://main:main@127.0.0.1:32773/test?sslmode=disable&charset=utf8
     KERNEL_CLASS='App\Kernel'
     APP_SECRET='$ecretf0rt3st'
     SYMFONY_DEPRECATIONS_HELPER=999999

.. index::
    single: Command;doctrine:fixtures:load

تحميل التركيبات لاختبار قاعدة البيانات:

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

التعامل مع النماذج في الاختبارات الوظيفية
-----------------------------------------------------------------------------

هل تريد الانتقال للمستوي التالي؟ جرب ان تقوم بإضافه تعليق وصورة علي مؤتمر من الاختبار عن طريق محاكاة تقديم نموذج ``Form submission``. يبدو طموح، اليس كذلك؟ تفقد الكود المطلوب: ليس اصعب من ما قمنا بكتابته مسبقا:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -29,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
             $this->assertSelectorExists('div:contains("There are 1 comments")');
         }
    +
    +    public function testCommentSubmission()
    +    {
    +        $client = static::createClient();
    +        $client->request('GET', '/conference/amsterdam-2019');
    +        $client->submitForm('Submit', [
    +            'comment_form[author]' => 'Fabien',
    +            'comment_form[text]' => 'Some feedback from an automated functional test',
    +            'comment_form[email]' => 'me@automat.ed',
    +            'comment_form[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
    +        ]);
    +        $this->assertResponseRedirects();
    +        $client->followRedirect();
    +        $this->assertSelectorExists('div:contains("There are 2 comments")');
    +    }
     }

لتقديم نموذج عن طريق ``submitForm()``، قم بإيجاد اسماء المدخلات بفضل ادوات التطوير الخاصه بالمتصفح او عن طريق لوحه النموذج من محلل سيمفوني (Symfony Profiler). لاحظ الاستخدام الذكي لإعادة استخدام صوره تحت البناء (Under construction)!

قم بتشغيل الإختبارات مرة أخري للتاكد من أن كل شئ أخضر:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

من أحد المميزات عند استخدام قاعدة البيانات الخاصة ببيئة "dev" للاختبارات هو أنه يمكنك فحص النتائج من المتصفح:

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

إعادة تحميل التركيبات
----------------------------------------

.. index::
    single: Command;doctrine:fixtures:load

إذا قمت بتشغيل الاختبارات مره اخري، ستفشل، حيث انه هناك اكثر من تعليق في قاعدة البيانات، المتإكد الذي يتحقق من عدد التعليقات يفشل. نحتاج ان نعيد حاله قاعدة البيانات كل مره نقوم بتشغيل الاختبارات عن طريق إعادة تشغيل التركيبات:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

أتمتة (Automating) خطوات العمل مع Makefile
-----------------------------------------------------------

.. index::
    single: Makefile

من المزعج تذكر خطوات تشغيل الإختبارت في كل مره. يجب توثيقها علي الأقل. ولكن التوثيق ينبغي ان يكون ملاذنا الأخير. بدلاً من ذلك، ما رايك ان نجعلها اتوماتيكه؟ سوف تخدم كتوثيق و ايضاً تساعد المطورين الاخريين، وتجعل حياة المطوريين اسهل و اسرع

.. index::
    single: Command;doctrine:fixtures:load

ان استخدام ``MakeFile`` طريقة واحده لجعل الأوامر اوتوماتيكية

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

لاحظ الـ ``-n`` في امر Doctrine، هو علم طبيعي في اوامر سيمفوني يجعلهم غير متفاعلين.

متي تريد ان تشغل الاختبارات، استخدم ``make tests``

.. code-block:: bash

    $ make tests

إعادة ضبت قاعدة البيانات بعد كل اختبار
----------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

إعادة ضبت قاعدة البيانات بعد كل اختبار امر لطيف، ولكن الاختبارات المستقله افضل. لا نريد ان يكون لدينا اختبار يعتمد علي نتائج اختباره اخر قبله. تغيير ترتيب الاختبارات لا ينبغي ان يغير النتائج. كما سنعرف الان. هذه ليست القضيه التي نريدها الان.

قم بنقل اختبار ``testConferencePage`` بعد ``testCommentSubmission``:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -15,21 +15,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }

    -    public function testConferencePage()
    -    {
    -        $client = static::createClient();
    -        $crawler = $client->request('GET', '/');
    -
    -        $this->assertCount(2, $crawler->filter('h4'));
    -
    -        $client->clickLink('View');
    -
    -        $this->assertPageTitleContains('Amsterdam');
    -        $this->assertResponseIsSuccessful();
    -        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    -        $this->assertSelectorExists('div:contains("There are 1 comments")');
    -    }
    -
         public function testCommentSubmission()
         {
             $client = static::createClient();
    @@ -44,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }
    +
    +    public function testConferencePage()
    +    {
    +        $client = static::createClient();
    +        $crawler = $client->request('GET', '/');
    +
    +        $this->assertCount(2, $crawler->filter('h4'));
    +
    +        $client->clickLink('View');
    +
    +        $this->assertPageTitleContains('Amsterdam');
    +        $this->assertResponseIsSuccessful();
    +        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    +        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +    }
     }

ستفشل الاختبارات الآن.

.. index::
    single: Doctrine;TestBundle

لتقوم بإعادة ضبت قاعدة البيانات بين الاختبارات، قم بتنصيب DoctrineTestBundle:

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

ستحتاج ان توافق علي تنصيب الوصفه (بما انها ليست حزمه مدعومه "رسمياً"):

.. code-block:: text
    :class: ignore

    Symfony operations: 1 recipe (d7f110145ba9f62430d1ad64d57ab069)
      -  WARNING  dama/doctrine-test-bundle (>=4.0): From github.com/symfony/recipes-contrib:master
        The recipe for this package comes from the "contrib" repository, which is open to community contributions.
        Review the recipe at https://github.com/symfony/recipes-contrib/tree/master/dama/doctrine-test-bundle/4.0

        Do you want to execute this recipe?
        [y] Yes
        [n] No
        [a] Yes for all packages, only for the current installation session
        [p] Yes permanently, never ask again for this project
        (defaults to n): p

تشغيل مستمع PHPUnit:

.. code-block:: diff
    :caption: patch_file

    --- a/phpunit.xml.dist
    +++ b/phpunit.xml.dist
    @@ -27,6 +27,10 @@
             </whitelist>
         </filter>

    +    <extensions>
    +        <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension" />
    +    </extensions>
    +
         <listeners>
             <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener" />
         </listeners>

و انتهينا.  اي تغييرات علي قاعدة البيانات اثناء الاختبارات تلقائيا يتم تجاهلها بعد الانتهاء من كل اختبار.

ينبغي ان تكون الاختبارات خضراء الآن:

.. code-block:: bash

    $ make tests

استخدام متصفح حقيقي للاختبارات الوظيفيه
--------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

الاختبارات الوظيفيه تستخدم متصفح مخصص يتصل بيسمفوني مباشرة. لكن يمكنك استخدام متصفح حقيقي يتصل بـ HTTP بفضل Panther من سيمفوني:

.. code-block:: bash

    $ symfony composer req panther --dev

يمكنك كتابه اختبارات تستخدم متصفح Google Chrome مع التغييرات الآتيه:

.. code-block:: diff
    :class: ignore

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -2,13 +2,13 @@

     namespace App\Tests\Controller;

    -use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    +use Symfony\Component\Panther\PantherTestCase;

    -class ConferenceControllerTest extends WebTestCase
    +class ConferenceControllerTest extends PantherTestCase
     {
         public function testIndex()
         {
    -        $client = static::createClient();
    +        $client = static::createPantherClient(['external_base_uri' => $_SERVER['SYMFONY_PROJECT_DEFAULT_ROUTE_URL']]);
             $client->request('GET', '/');

             $this->assertResponseIsSuccessful();

المتغير ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL``  يحتوي علي رابط الخادم المحلي.

تشغيل اختبارات الصندوق الأسود الوظيفية باستخدام Blackfire
---------------------------------------------------------------------------------------------------

طريقه اخري لتشغيل الاختبارات الوظيفيه عن طريق `Blackfire player <https://blackfire.io/player>`_. بالاضافه لما يمكنك فعله مع الاختبارات الوظيفيه. يمكنك  ان تؤدي اختبارات الاداء.

تفقد خطوه الاداء "Performance" لمعرفه المزيد.

.. sidebar:: الذهاب أبعد من ذلك

    * `قائمه بالتجديدات المعرفه في سيمفوني <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ للاختبارات الوظيفيه؛

    * `توثيقات PHPUnit <https://phpunit.de/documentation.html>`_؛

    * مكتبه التزوير `Faker <https://github.com/fzaninotto/Faker>`_ تولد بيانات حقيقة للتركبيات؛

    * `مستندات مكون الـ CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_؛

    * `سيمفوني بانثر <https://github.com/symfony/panther>`_ مكتبه لاختبار التصفح و استخراج بيانات الموقع في تطبيقات سيمفوني؛

    * توثيقات `Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
