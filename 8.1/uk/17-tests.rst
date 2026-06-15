Тестування
====================

.. index::
    single: PHPUnit

Оскільки ми починаємо додавати все більше і більше функціональності в застосунок, напевно, саме час поговорити про тестування.

*Цікавий факт*: я знайшов помилку під час написання тестів у цьому розділі.

Symfony використовує PHPUnit для модульного тестування. Встановімо його:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Написання модульних тестів
--------------------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

SpamChecker — це перший клас, для якого ми будемо писати тести. Згенеруйте модульний тест:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Тестування SpamChecker є складним завданням, оскільки ми, звичайно, не хочемо викликати API OpenAI: це було б повільно, дорого, а відповіді навіть не були б детермінованими. Ми збираємося замінити *платформу* на фіктивну.

.. index::
    single: Mock

Напишімо перший тест на той випадок, коли модель недоступна:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/SpamCheckerTest.php
    +++ w/tests/SpamCheckerTest.php
    @@ -2,12 +2,25 @@

     namespace App\Tests;

    +use App\Entity\Comment;
    +use App\SpamChecker;
     use PHPUnit\Framework\TestCase;
    +use Symfony\AI\Agent\Agent;
    +use Symfony\AI\Platform\Exception\RuntimeException;
    +use Symfony\AI\Platform\Test\InMemoryPlatform;

     class SpamCheckerTest extends TestCase
     {
    -    public function testSomething(): void
    +    public function testSpamScoreWhenTheModelIsDown(): void
         {
    -        $this->assertTrue(true);
    +        $comment = new Comment();
    +        $comment->setAuthor('Fabien');
    +        $comment->setEmail('fabien@example.com');
    +        $comment->setText('Such a nice conference!');
    +
    +        $platform = new InMemoryPlatform(fn () => throw new RuntimeException('The model is down.'));
    +        $checker = new SpamChecker(new Agent($platform, 'gpt-5-mini'));
    +
    +        $this->assertSame(1, $checker->getSpamScore($comment, []));
         }
     }

Клас ``InMemoryPlatform`` реалізує інтерфейс платформи, не викликаючи жодного зовнішнього API. Маючи об'єкт, який можна викликати (англ. callable), він може імітувати будь-яку поведінку, включно з помилками. Ми обгортаємо його в справжній ``Agent``, щоб логіка ``SpamChecker`` тестувалася по-справжньому.

Коли модель недоступна, коментарі мають потрапляти до модератора-людини: очікуваний результат — ``1``.

Виконайте тести, щоб перевірити, що вони проходять:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Додаймо тести для успішного сценарію:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/SpamCheckerTest.php
    +++ w/tests/SpamCheckerTest.php
    @@ -4,6 +4,7 @@ namespace App\Tests;

     use App\Entity\Comment;
     use App\SpamChecker;
    +use PHPUnit\Framework\Attributes\DataProvider;
     use PHPUnit\Framework\TestCase;
     use Symfony\AI\Agent\Agent;
     use Symfony\AI\Platform\Exception\RuntimeException;
    @@ -23,4 +24,25 @@ class SpamCheckerTest extends TestCase

             $this->assertSame(1, $checker->getSpamScore($comment, []));
         }
    +
    +    #[DataProvider('provideComments')]
    +    public function testSpamScore(int $expectedScore, string $answer): void
    +    {
    +        $comment = new Comment();
    +        $comment->setAuthor('Fabien');
    +        $comment->setEmail('fabien@example.com');
    +        $comment->setText('Such a nice conference!');
    +
    +        $platform = new InMemoryPlatform($answer);
    +        $checker = new SpamChecker(new Agent($platform, 'gpt-5-mini'));
    +
    +        $this->assertSame($expectedScore, $checker->getSpamScore($comment, []));
    +    }
    +
    +    public static function provideComments(): iterable
    +    {
    +        yield 'blatant_spam' => [2, 'blatant spam'];
    +        yield 'maybe_spam' => [1, 'Maybe spam.'];
    +        yield 'ham' => [0, 'ham'];
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

Тестування контролерів трохи відрізняється від тестування "звичайного" PHP-класу, оскільки ми хочемо виконати тести у контексті HTTP-запиту.

Створіть функціональний тест для контролера конференції:

.. code-block:: php
    :caption: tests/Controller/ConferenceControllerTest.php

    namespace App\Tests\Controller;

    use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

    class ConferenceControllerTest extends WebTestCase
    {
        public function testIndex(): void
        {
            $client = static::createClient();
            $client->request('GET', '/');

            $this->assertResponseIsSuccessful();
            $this->assertSelectorTextContains('h2', 'Give your feedback');
        }
    }

Використання ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` замість ``PHPUnit\Framework\TestCase`` в якості базового класу для наших тестів дає нам гарну абстракцію для функціональних тестів.

Змінна ``$client`` імітує браузер. Однак, замість того щоб робити HTTP-запит до сервера, вона працює із застосунком Symfony безпосередньо. Ця стратегія має кілька переваг: вона набагато швидша, ніж процес взаємодії між клієнтом і сервером, але крім цього вона також дозволяє тестам інтроспектувати стан сервісів після кожного HTTP-запиту.

Цей перший тест перевіряє, чи повертає головна сторінка HTTP-відповідь 200.

Твердження, такі як ``assertResponseIsSuccessful``, додані поверх PHPUnit, щоб полегшити вашу роботу. Існує багато таких тверджень, визначених Symfony.

.. tip::

    Ми використовували шлях ``/`` для URL-адреси замість того, щоб генерувати його через роутер. Це робиться спеціально, оскільки тестування URL-адрес кінцевих користувачів є частиною того, що ми хочемо протестувати. Якщо ви зміните шлях маршруту —  тести зламаються, як приємне нагадування про те, що ви, ймовірно, маєте переспрямувати стару URL-адресу на нову, щоб забезпечити взаємодію з пошуковими системами й веб-сайтами, які посилаються на ваш веб-сайт.

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
            ...

.. index:: Command;secrets:set

Щоб змусити тести працювати, ми маємо встановити секретний рядок ``OPENAI_API_KEY`` для цього ``test`` середовища:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Робота з тестовою базою даних
------------------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Як ми вже бачили, Symfony CLI автоматично надає змінну середовища ``DATABASE_URL``. Коли ``APP_ENV`` має значення ``test``, як це встановлено під час запуску PHPUnit, назва бази даних змінюється з ``app`` на ``app_test``, щоб тести мали свою власну базу даних:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Це дуже важливо, оскільки нам знадобляться стабільні дані для виконання наших тестів, і ми, звичайно, не хочемо перевизначати те, що ми зберегли в базі даних розробки.

Перш ніж виконати тест, нам потрібно "ініціалізувати" ``test`` базу даних (створити базу даних і виконати її міграцію):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Якщо ви зараз виконаєте тести, PHPUnit більше не буде взаємодіяти з вашою базою даних розробки. Щоб виконати тільки нові тести, передайте шлях до їх класу:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Якщо тест не проходить, може бути корисним проаналізувати об’єкт Response. Отримайте до нього доступ за допомогою ``$client->getResponse()`` та виведіть використовуючи ``echo``, щоб побачити, що він собою представляє.

Визначення фабрик
-----------------------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

Щоб мати змогу тестувати список коментарів, пагінацію та відправку форми, нам необхідно заповнити базу даних деякими даними. А щоб тести були незалежними один від одного, кожен тест має створювати саме той набір даних, який йому потрібен. *Фабрики об'єктів* (англ. object factories) — ідеальний інструмент для цього.

Встановіть Zenstruck Foundry:

.. code-block:: terminal

    $ symfony composer req foundry --dev

Згенеруйте фабрику для кожної сутності, яку потребують тести:

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

Фабрика описує, як побудувати дійсну сутність: для кожної властивості генерується значення за замовчуванням завдяки бібліотеці Faker. Створення об'єкта за допомогою фабрики також зберігає його. Налаштуйте значення за замовчуванням для конференції, щоб вони були більш реалістичними:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Factory/ConferenceFactory.php
    +++ w/src/Factory/ConferenceFactory.php
    @@ -34,9 +34,9 @@ final class ConferenceFactory extends PersistentObjectFactory
     protected function defaults(): array|callable
     {
         return [
    -        'city' => self::faker()->text(255),
    +        'city' => self::faker()->city(),
             'isInternational' => self::faker()->boolean(),
    -        'slug' => self::faker()->text(255),
    -        'year' => self::faker()->text(4),
    +        'slug' => '-',
    +        'year' => self::faker()->year(),
         ];
     }

Встановлення slug у ``-`` дозволяє слухачу сутності, який ми написали при додаванні slug-ів, обчислити справжнє значення: конференція, створена з містом ``Amsterdam`` і роком ``2019``, автоматично отримує slug ``amsterdam-2019``.

Зробіть те саме для коментарів:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Factory/CommentFactory.php
    +++ w/src/Factory/CommentFactory.php
    @@ -34,10 +34,10 @@ final class CommentFactory extends PersistentObjectFactory
     protected function defaults(): array|callable
     {
         return [
    -        'author' => self::faker()->text(255),
    +        'author' => self::faker()->name(),
             'conference' => ConferenceFactory::new(),
             'createdAt' => \DateTimeImmutable::createFromMutable(self::faker()->dateTime()),
    -        'email' => self::faker()->text(255),
    +        'email' => self::faker()->email(),
             'text' => self::faker()->text(),
         ];
     }

Зверніть увагу на значення за замовчуванням ``conference``: коли коментар створюється без явної конференції, Foundry створює її на льоту.

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

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -2,10 +2,17 @@

     namespace App\Tests\Controller;

    +use App\Factory\CommentFactory;
    +use App\Factory\ConferenceFactory;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    +use Zenstruck\Foundry\Test\Factories;
    +use Zenstruck\Foundry\Test\ResetDatabase;

     class ConferenceControllerTest extends WebTestCase
     {
    +    use Factories;
    +    use ResetDatabase;
    +
         public function testIndex(): void
         {
             $client = static::createClient();
    @@ -14,4 +21,24 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }
    +
    +    public function testConferencePage(): void
    +    {
    +        $client = static::createClient();
    +
    +        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    +        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $amsterdam]);
    +
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

Трейт ``Factories`` вмикає фабрики в тестах, а ``ResetDatabase`` скидає базу даних на початку кожного виконання тестів.

Опишімо, що відбувається в цьому тесті простою українською мовою:

* Тест створює саме той набір даних, який йому потрібен: дві конференції та один коментар, за допомогою фабрик;

* Як і в першому тесті, ми переходимо на головну сторінку;

* Метод ``request()`` повертає екземпляр ``Crawler``, який допомагає знайти елементи на сторінці (як-от посилання, форми чи все, що ви можете отримати за допомогою CSS селекторів або XPath);

* Завдяки селектору CSS ми стверджуємо, що в нас є дві конференції, перелічені на головній сторінці;

* Потім ми натискаємо на посилання "View" (оскільки неможливо натиснути більш ніж на одне посилання одночасно, Symfony автоматично вибирає перше знайдене);

* Ми перевіряємо назву сторінки, відповідь та заголовок ``<h2>``, щоб впевнитися, що ми знаходимося на потрібній сторінці (ми також могли б перевірити чи збігається запитуваний маршрут);

* Нарешті, ми стверджуємо, що на сторінці є 1 коментар. ``div:contains()`` не є валідним селектором CSS, але Symfony має кілька приємних доповнень, запозичених у jQuery.

Замість того щоб натискати на текст (тобто ``View``), ми могли б вибрати посилання за допомогою селектора CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Перевірте, що новий тест проходить успішно:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Відправка форми у функціональному тесті
--------------------------------------------------------------------------

Ви хочете вийти на наступний рівень? Спробуйте додати новий коментар до фотографії на сторінці конференції з тесту, імітуючи відправку форми. Це здається амбітним, чи не так? Подивіться на код, який нам потрібен: він не складніший за той, що ми вже писали:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -41,4 +41,23 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
             $this->assertSelectorExists('div:contains("There are 1 comments")');
         }
    +
    +    public function testCommentSubmission(): void
    +    {
    +        $client = static::createClient();
    +
    +        $berlin = ConferenceFactory::createOne(['city' => 'Berlin', 'year' => '2021', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $berlin]);
    +
    +        $client->request('GET', '/conference/berlin-2021');
    +        $client->submitForm('Submit', [
    +            'comment[author]' => 'Fabien',
    +            'comment[text]' => 'Some feedback from an automated functional test',
    +            'comment[email]' => 'me@automat.ed',
    +            'comment[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
    +        ]);
    +        $this->assertResponseRedirects();
    +        $client->followRedirect();
    +        $this->assertSelectorExists('div:contains("There are 2 comments")');
    +    }
     }

Щоб відправити форму за допомогою ``submitForm()``, знайдіть імена елементів за допомогою інструментів розробника у веб-браузері або вкладки Form на панелі Symfony Profiler. Зверніть увагу на те, як продумано повторно використовується зображення, яке вказує на те, що сайт знаходиться у розробці!

Виконайте тести ще раз, щоб перевірити, чи все проходить успішно:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Якщо ви хочете перевірити результат у браузері, зупиніть веб-сервер і повторно запустіть його для середовища ``test``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Повторне виконання тестів
-----------------------------------------------

Якщо ви виконаєте тести вдруге, вони все одно проходять: трейт ``ResetDatabase`` скидає базу даних на початку кожного виконання тестів, а кожен тест створює саме той набір даних, який йому потрібен. Немає спільного стану й залишків від попереднього виконання:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Автоматизація робочого процесу за допомогою Makefile
-------------------------------------------------------------------------------------------

.. index::
    single: Makefile

Необхідність запам'ятовувати послідовність команд для виконання тестів дратує. Це, принаймні, має бути задокументовано. Але документація — це крайній випадок. Натомість, як щодо автоматизації однотипних завдань? Це послужило б у якості документації, допомогло б іншим розробникам досліджувати проект та полегшити й прискорити розробку.

Використання ``Makefile`` є одним зі способів автоматизації команд:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:database:drop --force --env=test || true
    	symfony console doctrine:database:create --env=test
    	symfony console doctrine:migrations:migrate -n --env=test
    	symfony php bin/phpunit $(MAKECMDGOALS)
    .PHONY: tests

.. warning::

    У правилі Makefile відступ **має** складатися з одного символу табуляції замість пробілів.

Зверніть увагу на прапорець ``-n`` у команді Doctrine; це глобальний прапорець команд Symfony, що робить їх неінтерактивними.

Щоразу, коли ви хочете виконати тести, використовуйте ``make tests``:

.. code-block:: terminal

    $ make tests

Скидання бази даних після кожного тесту
-------------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Скидання бази даних після кожного виконання тестів, звісно,  чудово, але мати справді незалежні тести — ще краще. Ми не хочемо, щоб один тест спирався на результати попередніх. Зміна порядку проведення тестів не має призводити до зміни результату. Як ми зараз з'ясуємо, на даний момент це не так.

Помістіть тест ``testConferencePage`` після ``testCommentSubmission``:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -22,26 +22,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }

    -    public function testConferencePage(): void
    -    {
    -        $client = static::createClient();
    -
    -        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    -        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    -        CommentFactory::createOne(['conference' => $amsterdam]);
    -
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
         public function testCommentSubmission(): void
         {
             $client = static::createClient();
    @@ -41,5 +22,25 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseRedirects();
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }
    +
    +    public function testConferencePage(): void
    +    {
    +        $client = static::createClient();
    +
    +        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    +        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $amsterdam]);
    +
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

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

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

Готово. Будь-які зміни, внесені в тести, тепер автоматично скасовуються в кінці кожного тесту.

Тести знову мають проходити успішно:

.. code-block:: terminal

    $ make tests

Використання справжнього веб-браузера для функціональних тестів
------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Функціональні тести використовують спеціальний браузер, який працює безпосередньо з Symfony. Але ви також можете використовувати реальний веб-браузер і HTTP, завдяки Symfony Panther:

.. code-block:: terminal

    $ symfony composer req panther --dev

Потім ви можете написати тести, які використовують реальний браузер Google Chrome з наступними змінами:

.. code-block:: diff
    :class: ignore

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -2,13 +2,13 @@

     namespace App\Tests\Controller;

    -use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    +use Symfony\Component\Panther\PantherTestCase;

    -class ConferenceControllerTest extends WebTestCase
    +class ConferenceControllerTest extends PantherTestCase
     {
         public function testIndex(): void
         {
    -        $client = static::createClient();
    +        $client = static::createPantherClient(['external_base_uri' => rtrim($_SERVER['SYMFONY_PROJECT_DEFAULT_ROUTE_URL'], '/')]);
             $client->request('GET', '/');

             $this->assertResponseIsSuccessful();

Змінна середовища ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` містить URL-адресу локального веб-сервера.

Вибір правильного типу тесту
-----------------------------------------------------

.. index::
    single: Command;make:test

Наразі ми створили три різні типи тестів. Хоча ми використовували бандл Maker лише для створення класу модульного тесту, ми могли б використовувати його і для створення класів інших тестів:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

Бандл Maker підтримує генерування наступних типів тестів залежно від того, як ви хочете протестувати свій застосунок:

* ``TestCase``: базові тести PHPUnit;

* ``KernelTestCase``: базові тести, які мають доступ до сервісів Symfony;

* ``WebTestCase``: для виконання сценаріїв, подібних браузеру, але не виконуючих код JavaScript;

* ``ApiTestCase``: для виконання сценаріїв, орієнтованих на API;

* ``PantherTestCase``: для виконання сценаріїв e2e з використанням реального браузера чи HTTP-клієнта і реального веб-сервера.

Виконання функціональних тестів методом чорної скриньки за допомогою Blackfire
-------------------------------------------------------------------------------------------------------------------------------------------

Іншим способом виконання функціональних тестів є використання `Blackfire player`_. Крім функціонального тестування, цей інструмент також може проводити тестування продуктивності.

Перегляньте крок про :doc:`Продуктивність<29-performance>`, щоб дізнатися більше.

.. sidebar:: Йдемо далі

    * `Список тверджень, визначених Symfony`_ для функціональних тестів;

    * `Документація по PHPUnit`_;

    * `Документація Foundry`_;

    * `Бібліотека Faker`_ для генерування реалістичних даних фікстур;

    * `Документація по компоненту CssSelector`_;

    * `Symfony Panther`_ бібліотека для тестування у браузері та веб-сканування у застосунках Symfony;

    * `Документація по Make/Makefile`_.

.. _`Blackfire player`: https://blackfire.io/player
.. _`Список тверджень, визначених Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`Документація по PHPUnit`: https://phpunit.de/documentation.html
.. _`Документація Foundry`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`Бібліотека Faker`: https://github.com/FakerPHP/Faker
.. _`Документація по компоненту CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`Документація по Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
