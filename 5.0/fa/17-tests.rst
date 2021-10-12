آزمودن (Testing)
======================

.. index::
    single: PHPUnit

از آنجا که ما شروع به افزودن قابلیت‌های بیشتر و بیشتر در برنامه می‌کنیم‌‌، احتمالاً زمان مناسبی برای صحبت در‌باره‌ی آزمودن اپلیکیشن می‌باشد.

*حقیقت خنده‌دار*: من هنگام نوشتن آزمون‌ها در این فصل، یک باگ پیدا کردم.

سیمفونی برای آزمون‌های واحد (unit test)، به PHPUnit تکیه دارد. بیایید آن را نصب کنیم:

.. code-block:: bash

    $ symfony composer req phpunit --dev

نوشتن آزمون‌های واحد (Unit Tests)
----------------------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

`SpamChecker` اولین کلاسی است که می‌خواهیم برای آن آزمون بنویسیم. یک آزمون واحد تولید کنید:

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

آزمودن SpamChecker یک چالش است زیرا مطمئناً نمی‌خواهیم واقعاً API مربوط به Akismet را فراخوانی کنیم. می‌خواهیم API را  *تقلید (mock)* کنیم.

.. index::
    single: Mock

بیایید اولین آزمون را برای زمانی که API یک خطا بازمی‌گرداند، بنویسیم:

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

کلاس ``MockHttpClient`` این امکان را می‌دهد تا هر سرور HTTP را تقلید کنیم. این کلاس آرایه‌ی از نمونه‌های ``MockResponse`` را که حاوی بدنه (body) و سربرگ‌های (headers) پاسخ مورد نظر هستند، می‌گیرد.

سپس متد ``getSpamScore()``را فراخوانی می‌کنیم و بررسی می‌کنیم که آیا یک استثناء توسط متد ``getSpamScore()`` در PHPUnit پرتاب می‌شود یا خیر.

برای بررسی قبولی در آزمون، آزمون را اجرا کنید:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

بیایید برای مسیر خوشحال، آزمون اضافه کنیم:

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

فراهم‌کنندگان داده در PHPUnit، به ما اجازه می‌دهد تا یک منطق آزمون یکسان را برای مورد آزمون‌های (test case) مختلف مجدداً استفاده کنیم.

نوشتن آزمون‌های کارکردی (Functional Tests) برای کنترلرها
------------------------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

آزمودن کنترلرها نسبت به آزمودن کلاس‌های PHP «معمولی»، کمی متفاوت است. زیرا که می‌خواهیم آن‌ها را در زمینه‌ی درخواست‌های HTTP اجرا کنیم.

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

یک آزمون کارکردی برای کنترلر Conference ایجاد کنید:

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

اولین آزمون بررسی می‌کند که صفحه‌ی اصلی، پاسخ HTTP از نوع ۲۰۰ باز می‌گرداند یا خیر.

متغیر ``$client`` یک مرورگر را شبیه‌سازی می‌کند. البته به جای فراخوانی HTTP به سرور، اپلیکیشن سیمفونی را به صورت مستقیم فراخوانی می‌کند. این راهبرد مزایای متعددی دارد: در مقایسه با رفت و برگشت میان کلاینت و سرور سریع‌تر است، همچنین اجازه می‌دهد تا آزمون‌ها، وضعیت سرویس‌ها را پس از هر درخواست HTTP بازرسی کنند.

ادعاهایی (Assertions) همچون ``assertResponseIsSuccessful`` بر روی PHPUnit اضافه شده‌اند تا کار شما را راحت کنند. تعداد زیادی ادعا توسط سیمفونی تعریف شده است.

.. tip::

    ما از ``/`` به عنوان URL استفاده کردیم به جای اینکه از طریق راه‌یاب (router) آن را تولید کنیم. هدف و علت این کار آن است که URLهای کاربر نهایی نیز بخشی از چیزی است که می‌خواهیم آزموده شود. اگر شما مسیرِ راه (route path) را تغییر دهید، آزمون شکست می‌خورد و به خوبی یادآور می‌شود که شما احتمالاً باید URL قدیمی را به URL جدید بازهدایت کنید تا برای موتورهای جستجو و وب‌سایت‌هایی که به وب‌سایت شما پیوند می‌دهند، مشکلی پیش نیاید.

.. note::

    می‌توانستیم آزمون را به کمک باندلِ maker تولید کنیم:

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

آزمون‌های PHPUnit در محیط اختصاصی ``test`` اجرا می‌شوند. ما باید رمز ``AKISMET_KEY`` را برای این محیط تنظیم کنیم:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    زمانی که آزمون شکست می‌خورد، ممکن است بازرسی شیء پاسخ مفید واقع شود. از طریق ``$client->getResponse()`` و ``echo`` به آن دست پیدا کنید تا ببینید به چه شکل است.

تعریف Fixture‌ها
-------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

برای اینکه قادر به آزمودن لیست کامنت‌ها، صفحه‌بندی و فرم ارسال باشیم، ما نیاز داریم تا پایگاه‌داده را با مقداری داده پر کنیم. همچنین می‌خواهیم که این داده‌ها بین آزمونهای مختلف یکسان باشد تا موجب قبولی آزمون شود. Fixtureها دقیقاً چیزی هستند که نیازشان داریم.

باندل Doctrine Fixtures را نصب کنید:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

در هنگام نصب، پوشه‌ی جدید ``src/DataFixtures/`` به همراه یک کلاس نمونه ایجاد شده که آماده‌ی شخصی‌سازی است. فعلاً ۲ کنفرانس و یک کامنت به آن اضافه کنید:

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

زمانی که fixtureها را بار بگیریم، تمام داده‌ها پاک خواهد شد؛ از جمله کاربر مدیر. بیایید برای جلوگیری از این امر، کاربر مدیر را به fixtureها اضافه کنیم:

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

    اگر به خاطر نمی‌آورید که کدام سرویس را برای انجام یک وظیفه نیاز دارید، از فرمان ``debug:autowiring`` به همراه یک کلیدواژه استفاده کنید:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

بارگرفتن Fixtureها
----------------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

Crawl کردن یک وب‌سایت در آزمون‌های کارکردی
---------------------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

همانطور که دیدیم، HTTP client در آزمون‌ها برای شبیه‌سازی مرورگر استفاده می‌شود. بنابراین می‌توانیم درون یک وب‌سایت پیمایش کنیم انگار که در حال استفاده از یک مرورگر بی‌سر (headless) هستیم.

یک آزمون جدید بیافزایید که از درون صفحه‌ی اصلی، بر روی صفحه‌ی یک کنفرانس کلیک می‌کند:

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

بیایید به فارسی تشریح کنیم که چه اتفاقی درون این آزمون می‌افتد:

* همچون آزمون اول، به صفحه‌ی اصلی می‌رویم؛

* متد ``request()`` یک نمونه ``Crawler`` بازمی‌گرداند که کمک می‌کند تا المان‌های درون صفحه را پیدا کنیم (مثل پیوندها، فرم‌ها، یا هرچیزی که بتوان از طریق انتخابگرهای CSS یا XPath به آن دست یافت)؛

* به لطف انتخابگر CSS (CSS selector)، ما ادعا می‌کنیم که ۲ کنفرانس در صفحه‌ی اصلی لیست شده است؛

* سپس بر روی پیوند «View» کلیک می‌کنیم (از آنجایی که سیمفونی نمی‌تواند در آن واحد بر روی بیش از یک پیوند کلیک نماید، بر روی اولین موردی که پیدا کند را کلیک می‌کند)؛

* عنوان صفحه، پاسخ و جزء ``<h2>`` صفحه را ادعا می‌کنیم تا اطمینان یابیم در صفحه‌ی درست قرار داریم (همچنین می‌توانیم راه -route- را بررسی کنیم که مطابقت داشته باشد)؛

* در نهایت، ادعا می‌کنیم که یک کامنت در صفحه وجود دارد. ``div:contains()`` یک انتخابگر معتبر CSS نیست، اما سیمفونی تعدادی افزودنی خوب دارد که آن‌ها را از jQuery قرض گرفته است.

به جای کلیک کردن بر روی متن (به عبارت دیگر همان ``View``)، می‌توانستیم پیوند را از طریق انتخابگر CSS هم انتخاب کنیم:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

بررسی کنید که آزمون جدید سبز است:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

کارکردن با یک پایگاه‌داده‌ی آزمون
----------------------------------------------------------------

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

بارگرفتن fixtureها برای محیط/پایگاه‌داده‌ی ``test``:

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

ارسال یک فرم در آزمون کارکردی
-----------------------------------------------------

می‌خواهید به مرحله‌ی بعد برسید؟ سعی کنید از طریق یک آزمون و شبیه‌سازی ارسال یک فرم، کامنتی جدید را به همراه یک عکس از کنفرانس اضافه کنید. این کار به نظر بلندپروازانه می‌آید، اینطور نیست؟ به کد مورد نیاز نگاه کنید، پیچیده‌تر از چیزی که تا الان نوشته‌ایم نیست:

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

برای ارسال فرم از طریق ``submitForm()``، اسم ورودی‌ها را به کمک ابزار‌های  توسعه‌ای مرورگر یا از طریق پنل فرم در نمایه‌ساز سیمفونی، پیدا کنید. به بازاستفاده‌ی زیرکانه از تصویر دردست احداث توجه کنید!

برای بررسی سبز بودن همه چیز، آزمون‌ها را مجدداً اجرا کنید:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

یک مزیت استفاده از پایگاه‌داده‌ی «محیط توسعه» برای آزمون‌ها این است که می‌توانید نتایج را در مرورگر بررسی کنید:

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

بارگرفتن مجدد Fixtureها
-------------------------------------

.. index::
    single: Command;doctrine:fixtures:load

اگر آزمون‌ها را برای بار دوم اجرا کنید، آن‌ها باید شکست بخورند. از آنجایی که در حال حاضر کامنت‌های بیشتری در پایگاه‌داده وجود دارد، ادعایی که تعداد کامنت‌ها را بررسی می‌کند، خراب شده است. ما نیاز داریم که قبل از هر اجرا، وضعیت پایگاه‌داده را از طریق بارگیری مجدد Fixtureها، بازتنظیم کنیم:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

خودکارسازی جریان‌کارتان از طریق یک Makefile
--------------------------------------------------------------------------

.. index::
    single: Makefile

اجبار در به خاطر سپردن دنباله‌ای از فرامین برای اجرای آزمون‌ها، آزاردهنده است. این موضوع حداقل باید مستند شود. اما مستندسازی باید آخرین گزینه‌ باشد. به جای اینکار، نظرتان در مورد خودکارسازی فعالیت‌های روزمره چیست؟ این روش می‌تواند کار مستندسازی را انجام داده، به کشف این موضوع توسط سایر توسعه‌دهندگان کمک کرده و زندگی توسعه‌دهندگان را ساده‌تر و سریع‌تر کند.

.. index::
    single: Command;doctrine:fixtures:load

استفاده از `Makefile`` یک راه برای خودکارسازی فرامین است:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

به پرچم ``-n`` در فرمان Doctrine توجه کنید؛ این یک پرچم جهانی برای فرمان‌های سیمفونی است که آن‌ها را غیر تعاملی می‌کند.

هر زمان که می‌خواهید آزمون‌ها را اجرا کنید، از ``make tests`` استفاده کنید:

.. code-block:: bash

    $ make tests

بازتنظیم پایگاه‌داده پس از هر آزمون
------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

بازتنظیم پایگاه‌داده پس از هربار اجرای آزمون خوب است، اما داشتن آزمون‌های واقعاً مستقل از آن هم بهتر است. ما نمی‌خواهیم که یک آزمون وابسته به نتایج آزمون‌های قبلی باشد. تغییر ترتیب اجرای آزمون‌ها نباید حاصل کار را تغییر دهد. همانطور که خواهیم دید، در حال حاضر اینگونه نیست.

آزمون ``testConferencePage`` را پس از آزمون ``testCommentSubmission`` قرار دهید:

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

حالا آزمون‌ها شکست می‌خورند.

.. index::
    single: Doctrine;TestBundle

برای بازتنظیم پایگاه‌داده میان آزمون‌ها، DoctrineTestBundle را نصب کنید:

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

نیاز دارید که اجرای recipe را تأیید کنید (زیرا این یک باندل با پشتیبانی «رسمی» نیست):

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

شنونده‌ی PHPUnit را فعال کنید:

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

و تمام. هر تغییری که در طول اجرای آزمون انجام شود، پس از پایان هر آزمون به صورت خودکار به حالت اول برمی‌گردد.

آزمون‌ها باید مجدداً سبز باشند:

.. code-block:: bash

    $ make tests

استفاده از یک مرورگر واقعی برای آزمون‌های کارکردی
--------------------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

آزمون‌های کارکردی، از یک مرورگر ویژه که لایه‌ی سیمفونی را مستقیماً فراخوانی می‌کند، استفاده می‌کنند. اما شما به لطف سیمفونی Panther، می‌توانید از یک مرورگر و لایه‌ی HTTP واقعی استفاده کنید:

.. code-block:: bash

    $ symfony composer req panther --dev

پس از این شما با انجام تغییرات زیر می‌توانید آزمون‌هایی بنویسید که از یک مرورگر گوگل کروم واقعی استفاده می‌کنند:

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

The ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` environment variable contains the URL of the local web server.

اجرای آزمون‌های کارکردی جعبه سیاه با Blackfire
------------------------------------------------------------------------------

یکی دیگر از راه‌های اجرای آزمون‌های کارکردی، استفاده از `Blackfire player <https://blackfire.io/player>`_ است. علاوه بر کارهایی که می‌توانید با آزمون‌های کارکردی انجام دهید، این روش می‌تواند آزمون‌های کارایی (performance tests) را نیز انجام دهد.

برای یادگیری بیشتر، به گام مربوط به «کارایی (Performance)» رجوع کنید.

.. sidebar:: بیشتر بدانید

    * `لیست ادعاهای تعریف‌شده توسط سیمفونی <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ برای آزمون‌های کارکردی؛

    * `مستندات PHPUnit <https://phpunit.de/documentation.html>`_؛

    * `کتابخانه‌ی Faker <https://github.com/fzaninotto/Faker>`_ برای تولید داده‌های fixture به صورت واقع‌بینانه؛

    * `مستندات کامپوننت CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_؛

    * کتابخانه‌ی `سیمفونی Panther <https://github.com/symfony/panther>`_ برای آزمودن از طریق مرورگر و crawl کردن وب در اپلیکیشن‌های سیمفونی؛

    * `مستندات Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
