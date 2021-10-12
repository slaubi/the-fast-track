Тестування
====================

.. index::
    single: PHPUnit

Оскільки ми починаємо додавати все більше і більше функціональності в застосунок, напевно, саме час поговорити про тестування.

*Цікавий факт*: я знайшов помилку під час написання тестів у цьому розділі.

Symfony використовує PHPUnit для модульного тестування. Встановімо його:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Написання модульних тестів
--------------------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

SpamChecker — це перший клас, для якого ми будемо писати тести. Згенеруйте модульний тест:

.. code-block:: bash

    $ symfony console make:test TestCase SpamCheckerTest

Тестування SpamChecker є складним завданням, оскільки ми, звичайно, не хочемо викликати API Akismet. Ми збираємося *імітувати* API.

.. index::
    single: Mock

Напишімо перший тест на той випадок, коли API повертає помилку:

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
    -    public function testSomething(): void
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

Клас ``MockHttpClient`` дозволяє імітувати будь-який HTTP-сервер. Він приймає масив екземплярів ``MockResponse``, що містять очікуване тіло і заголовки відповіді.

Потім ми викликаємо метод ``getSpamScore()`` і перевіряємо чи було кинуто виняток, за допомогою методу PHPUnit ``expectException()``.

Виконайте тести, щоб перевірити, що вони проходять:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Додаймо тести для успішного сценарію:

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

Постачальники даних PHPUnit дозволяють повторно використовувати ту саму логіку тестування для кількох тестових випадків.

Написання функціональних тестів для контролерів
------------------------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Тестування контролерів трохи відрізняється від тестування "звичайного" PHP-класу, оскільки ми хочемо виконати тести у контексті HTTP-запиту.

Створіть функціональний тест для контролера конференції:

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

Використання ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` замість ``PHPUnit\Framework\TestCase`` в якості базового класу для наших тестів дає нам гарну абстракцію для функціональних тестів.

Змінна ``$client`` імітує браузер. Однак замість того, щоб робити HTTP-запит до сервера, вона працює із застосунком Symfony безпосередньо. Ця стратегія має кілька переваг: вона набагато швидша, ніж процес взаємодії між клієнтом і сервером, але крім цього вона також дозволяє тестам інтроспектувати стан сервісів після кожного HTTP-запиту.

Цей перший тест перевіряє, чи повертає головна сторінка HTTP-відповідь 200.

Твердження, такі як ``assertResponseIsSuccessful``, додані поверх PHPUnit, щоб полегшити вашу роботу. Існує багато таких тверджень, визначених Symfony.

.. tip::

    Ми використовували шлях ``/`` для URL-адреси замість того, щоб генерувати його через роутер. Це робиться спеціально, оскільки тестування URL-адрес кінцевих користувачів є частиною того, що ми хочемо протестувати. Якщо ви зміните шлях маршруту —  тести зламаються, як приємне нагадування про те, що ви, ймовірно, маєте перенаправити стару URL-адресу на нову, щоб забезпечити взаємодію з пошуковими системами й веб-сайтами, які посилаються на ваш веб-сайт.

.. note::

    Ми могли б згенерувати тест за допомогою бандла Maker:

    .. code-block:: bash

        $ symfony console make:test WebTestCase Controller\\ConferenceController

Налаштування тестового середовища
----------------------------------------------------------------

.. index::
    single: Symfony Environments

За замовчуванням тести PHPUnit виконуються у середовищі Symfony ``test``, як визначено в конфігураційному файлі PHPUnit:

.. code-block:: xml
    :caption: phpunit.xml.dist
    :emphasize-lines: 4
    :class: ignore

    <phpunit>
        <php>
            <ini name="error_reporting" value="-1" />
            <server name="APP_ENV" value="test" force="true" />
            <server name="SHELL_VERBOSITY" value="-1" />
            <server name="SYMFONY_PHPUNIT_REMOVE" value="" />
            <server name="SYMFONY_PHPUNIT_VERSION" value="8.5" />
        </php>
    </phpunit>

.. index:: Command;secrets:set

Щоб змусити тести працювати, ми маємо встановити секретний рядок ``AKISMET_KEY`` для цього ``test`` середовища:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

.. note::

    Як видно з попереднього розділу, ``APP_ENV=test`` означає, що змінну середовища ``APP_ENV`` встановлено для контексту команди. У Windows використовуйте ``--env=test`` замість ``symfony console secrets:set AKISMET_KEY --env=test``

Робота з тестовою базою даних
------------------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Як ми вже бачили, CLI Symfony автоматично надає змінну середовища ``DATABASE_URL``. Коли ``APP_ENV`` — ``test``, як встановлено під час запуску PHPUnit, це змінює ім'я бази даних з ``main`` на ``main_test``, щоб тести мали власну базу даних. Це дуже важливо, оскільки нам потрібні стабільні дані для виконання наших тестів, і ми, звичайно, не хочемо змінювати те, що ми зберігаємо в продакшн базі даних.

Перш ніж виконати тест, нам потрібно "ініціалізувати" ``test`` базу даних (створити базу даних і виконати її міграцію):

.. code-block:: bash

    $ APP_ENV=test symfony console doctrine:database:create
    $ APP_ENV=test symfony console doctrine:migrations:migrate -n

Якщо ви зараз виконаєте тести, PHPUnit більше не буде взаємодіяти з вашою продакшн базою даних. Щоб виконати тільки нові тести, передайте шлях до їх класу:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Зверніть увагу, що ми явно встановлюємо ``APP_ENV`` навіть під час запуску PHPUnit, щоб дозволити Symfony CLI встановити ім'я бази даних у ``main_test``.

.. tip::

    Якщо тест не проходить, може бути корисним проаналізувати об’єкт Response. Отримайте до нього доступ за допомогою ``$client->getResponse()`` та виведіть використовуючи ``echo``, щоб побачити, що він собою представляє.

Визначення фікстур
-----------------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Щоб мати змогу тестувати список коментарів, пагінацію та відправку форми, нам необхідно заповнити базу даних деякими даними. Ми хочемо, щоб дані були однаковими між виконанням тестів, щоб тести проходили успішно. Фікстури — це саме те, що нам потрібно.

Встановіть бандл Doctrine Fixtures:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Під час встановлення було створено новий каталог ``src/DataFixtures/`` зі зразком класу, готовим до налаштування. Поки що додайте дві конференції та один коментар:

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

Коли ми завантажимо фікстури, всі дані будуть видалені; включаючи користувача адміністратора. Щоб уникнути цього, додаймо користувача адміністратора до фікстур:

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

    Якщо ви не пам'ятаєте, який сервіс потрібно використовувати для даного завдання, використовуйте ``debug:autowiring`` з певним ключовим словом:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Завантаження фікстур
---------------------------------------

.. index:: ! Command;doctrine:fixtures:load

Завантажте фікстури для середовища/бази даних ``test``:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load

Сканування веб-сайту у функціональних тестах
-----------------------------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Як ми вже бачили, HTTP-клієнт, який використовується у тестах, імітує браузер, тому ми можемо переміщатися по веб-сайту так, ніби ми використовуємо браузер без графічного інтерфейсу.

Додайте новий тест, який натискає на сторінку конференції з головної сторінки:

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

Опишімо, що відбувається в цьому тесті простою українською мовою:

* Як і в першому тесті, ми переходимо на головну сторінку;

* Метод ``request()`` повертає екземпляр ``Crawler``, який допомагає знайти елементи на сторінці (наприклад, посилання, форми або все, що ви можете отримати за допомогою CSS селекторів чи XPath);

* Завдяки селектору CSS ми стверджуємо, що в нас є дві конференції, перелічені на головній сторінці;

* Потім ми натискаємо на посилання "View" (оскільки неможливо натиснути більш ніж на одне посилання одночасно, Symfony автоматично вибирає перше знайдене);

* Ми перевіряємо назву сторінки, відповідь та заголовок ``<h2>``, щоб впевнитися, що ми знаходимося на потрібній сторінці (ми також могли б перевірити чи збігається запитуваний маршрут);

* Нарешті, ми стверджуємо, що на сторінці є 1 коментар. ``div:contains()`` не є валідним селектором CSS, але Symfony має кілька приємних доповнень, запозичених у jQuery.

Замість того, щоб натискати на текст (тобто ``View``), ми могли б вибрати посилання за допомогою селектора CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Перевірте, що новий тест проходить успішно:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Відправка форми у функціональному тесті
--------------------------------------------------------------------------

Ви хочете вийти на наступний рівень? Спробуйте додати новий коментар до фотографії на сторінці конференції з тесту, імітуючи відправку форми. Це здається амбітним, чи не так? Подивіться на код, який нам потрібен: він не складніший за той, що ми вже писали:

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

Щоб відправити форму за допомогою ``submitForm()``, знайдіть імена елементів за допомогою інструментів розробника у веб-браузері або вкладки Form на панелі Symfony Profiler. Зверніть увагу на те, як продумано повторно використовується зображення, яке вказує на те, що сайт знаходиться у розробці!

Виконайте тести ще раз, щоб перевірити, чи все проходить успішно:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Якщо ви хочете перевірити результат у браузері, зупиніть веб-сервер і повторно запустіть його для середовища ``test``:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Перезавантаження фікстур
-----------------------------------------------

.. index::
    single: Command;doctrine:fixtures:load

Якщо ви виконуєте тести вдруге, вони не пройдуть успішно. Оскільки в базі даних тепер більше коментарів, твердження, що перевіряє їх кількість, порушено. Нам потрібно скидати стан бази даних між кожним виконанням, шляхом перезавантаження фікстур:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load
    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Автоматизація робочого процесу за допомогою Makefile
-------------------------------------------------------------------------------------------

.. index::
    single: Makefile

Необхідність запам'ятовувати послідовність команд для виконання тестів дратує. Це, принаймні, має бути задокументовано. Але документація — це крайній випадок. Натомість, як щодо автоматизації однотипних завдань? Це послужило б у якості документації, допомогло б іншим розробникам досліджувати проект та полегшити й прискорити розробку.

.. index::
    single: Command;doctrine:fixtures:load

Використання ``Makefile`` є одним зі способів автоматизації команд:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests: export APP_ENV=test
    tests:
    	symfony console doctrine:database:drop --force || true
    	symfony console doctrine:database:create
    	symfony console doctrine:migrations:migrate -n
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit $@
    .PHONY: tests

.. warning::

    У правилі Makefile відступ **має** складатися з одного символу табуляції замість пробілів.

Зверніть увагу на прапорець ``-n`` у команді Doctrine; це глобальний прапорець команд Symfony, що робить їх неінтерактивними.

Щоразу, коли ви хочете виконати тести, використовуйте ``make tests``:

.. code-block:: bash

    $ make tests

Скидання бази даних після кожного тесту
-------------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Скидання бази даних після кожного виконання тестів, звісно,  чудово, але мати справді незалежні тести — ще краще. Ми не хочемо, щоб один тест спирався на результати попередніх. Зміна порядку проведення тестів не має призводити до зміни результату. Як ми зараз з'ясуємо, на даний момент це не так.

Помістіть тест ``testConferencePage`` після ``testCommentSubmission``:

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

Тепер тести не проходять успішно.

.. index::
    single: Doctrine;TestBundle

Щоб скидати базу даних між тестами, встановіть DoctrineTestBundle:

.. code-block:: bash
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: bash

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Вам потрібно буде підтвердити виконання рецепту (тому, що він не є "офіційно" підтримуваним бандлом):

.. code-block:: text
    :class: ignore

    Symfony operations: 1 recipe (a5c79a9ff21bc3ae26d9bb25f1262ed7)
      -  WARNING  dama/doctrine-test-bundle (>=4.0): From github.com/symfony/recipes-contrib:master
        The recipe for this package comes from the "contrib" repository, which is open to community contributions.
        Review the recipe at https://github.com/symfony/recipes-contrib/tree/master/dama/doctrine-test-bundle/4.0

        Do you want to execute this recipe?
        [y] Yes
        [n] No
        [a] Yes for all packages, only for the current installation session
        [p] Yes permanently, never ask again for this project
        (defaults to n): p

Увімкніть слухача PHPUnit:

.. code-block:: diff
    :caption: patch_file

    --- a/phpunit.xml.dist
    +++ b/phpunit.xml.dist
    @@ -29,6 +29,10 @@
             </include>
         </coverage>

    +    <extensions>
    +        <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension" />
    +    </extensions>
    +
         <listeners>
             <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener" />
         </listeners>

Готово. Будь-які зміни, внесені в тести, тепер автоматично скасовуються в кінці кожного тесту.

Тести знову мають проходити успішно:

.. code-block:: bash

    $ make tests

Використання справжнього веб-браузера для функціональних тестів
------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Функціональні тести використовують спеціальний браузер, який працює безпосередньо з Symfony. Але ви також можете використовувати реальний веб-браузер і HTTP, завдяки Symfony Panther:

.. code-block:: bash

    $ symfony composer req panther --dev

Потім ви можете написати тести, які використовують реальний браузер Google Chrome з наступними змінами:

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

Змінна середовища ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` містить URL-адресу локального веб-сервера.

Виконання функціональних тестів методом чорної скриньки за допомогою Blackfire
-------------------------------------------------------------------------------------------------------------------------------------------

Іншим способом виконання функціональних тестів є використання `Blackfire player <https://blackfire.io/player>`_. Крім функціонального тестування, цей інструмент також може проводити тестування продуктивності.

Щоб дізнатися більше, перегляньте крок про "Продуктивність".

.. sidebar:: Йдемо далі

    * `Список тверджень, визначених Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ для функціональних тестів;

    * `Документація по PHPUnit <https://phpunit.de/documentation.html>`_;

    * `Бібліотека Faker <https://github.com/FakerPHP/Faker>`_ для генерування реалістичних даних фікстур;

    * `Документація по компоненту CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * `Symfony Panther <https://github.com/symfony/panther>`_ бібліотека для тестування у браузері та веб-сканування у застосунках Symfony;

    * `Документація по Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
