Тестирование
========================

.. index::
    single: PHPUnit

Поскольку мы всё больше и больше добавляем новой функциональности в приложение, наверное, сейчас самое время обсудить тестирование.

*Забавный факт*: я нашёл баг во время написания тестов в этой главе.

Symfony использует PHPUnit для модульного тестирования. Давайте установим его:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Написание модульных тестов
--------------------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

Давайте начнём с тестирования класса ``SpamChecker``. Создайте образец теста для него:

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

Протестировать SpamChecker непросто, потому что этот класс взаимодействует с удалённым API-интерфейсом Akismet, который нам явно не стоит использовать во время тестирования. Поэтому будем *имитировать* работу этого API.

.. index::
    single: Mock

Давайте напишем наш первый тест для случая, когда API возвращает ошибку:

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

С помощью класса ``MockHttpClient`` можно создать фиктивную реализацию любого HTTP-сервера. Для этого классу нужно передать массив с экземплярами ``MockResponse``, каждый из которых содержит ожидаемые тело и заголовки ответа.

Затем вызываем метод ``getSpamScore()`` и проверяем, что было выброшено исключение, используя встроенный в PHPUnit метод ``expectException()``.

Запустите тесты и посмотрите, что все они успешно выполнились:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Давайте добавим тесты на позитивный сценарий:

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

В PHPUnit есть провайдеры данных, при помощи которых в разных тестах можно использовать одни и те же тестовые данные.

Написание функциональных тестов для контроллеров
--------------------------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Тестирование контроллеров несколько отличается от тестирования "обычного" PHP-класса, поскольку процесс происходит в контексте HTTP-запроса.

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

Создайте функциональный тест для контроллера конференций:

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

Первый тест проверяет, что главная страница возвращает статус 200 в HTTP-ответе.

Переменная ``$client`` предназначена для имитации браузера. Вместо отправки HTTP-запросов на сервер, происходит обращение к приложению Symfony напрямую. Такой подход имеет свои преимущества: он намного быстрее реального взаимодействия между клиентом и сервером, а ещё позволяет контролировать состояние сервисов после каждого HTTP-запроса.

Такие проверки, как ``assertResponseIsSuccessful``, добавлены поверх PHPUnit, чтобы упростить вам работу. В Symfony ещё много подобных проверок.

.. tip::

    Мы использовали ``/`` в качестве URL-адреса, вместо получения его через маршрутизатор. Это сделано не случайно, так как тестирование URL-адресов, используемых пользователем,  являются частью того, что мы хотим протестировать. Если вы поменяете адрес, тесты уже не будут успешно выполняться. Это послужит полезным напоминанием о том, что вам, скорее всего, нужно сделать редирект со старого адреса на новый, чтобы поисковые машины и ссылающиеся сайты узнали об этом.

.. note::

    Создадим тест с помощью бандла maker:

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

Тесты PHPUnit выполняются в отдельном окружении ``test``. Нам нужно создать переменную с секретным ключом ``AKISMET_KEY`` для этого окружения:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Если тест не проходит, имеет смысл посмотреть на состояние объекта Response — вывести на экран результат выполнения метода ``$client->getResponse()`` через конструкцию ``echo``.

Определение фикстур
-------------------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Для тестирования списка комментариев, пагинации и отправки формы нужно заполнить базу данных какими-нибудь данными. При этом немаловажно поддерживать такие данные однородными между запусками тестов. И вот тут приходят на помощь фикстуры.

Установите бандл для фикстур в Doctrine:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Во время установки была создана новая директория ``src/DataFixtures/`` вместе с примером класса, который можно редактировать как угодно. А пока добавьте в него две конференции и один комментарий:

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

После применения фикстур все ранее сохранённые данные будут удалены, включая пользователя-администратора. Чтобы этого не произошло, добавим его в фикстуры:

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

    Если вы не помните, какой сервис вам нужно использовать для выполнения какой-либо задачи, воспользуетесь командой ``debug:autowiring``, указав в качестве аргумента ключевое слово для поиска:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Применение фикстур
-----------------------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

Сканирование сайта в функциональных тестах
--------------------------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Как мы уже видели, используемый в тестах HTTP-клиент имитирует работу браузера. Таким образом, можно переходить по страницам сайта, как при использовании браузера без графического интерфейса.

Напишите новый тест, который щёлкает по любой ссылке конференции на главной странице:

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

Давайте опишем, какие действия должны происходить в тесте  на простом английском языке:

* Как и в первом тесте, сначала переходим на главную страницу;

* Метод ``request()`` возвращает экземпляр ``Crawler``, с помощью которого можно найти нужные элементы на странице (например, ссылки, формы и всё остальное, что можно получить через селекторы CSS или XPath);

* Благодаря CSS-селектору проверяем, что на главной странице есть две конференции;

* Далее кликаем по ссылке "View" (из-за того, что один вызов щёлкает только по одной ссылке, Symfony автоматически выберет первую найденную в разметке);

* Проверяем заголовок страницы, ответ и тег ``<h2>`` для того, чтобы знать наверняка, что мы находимся на нужной странице (дополнительно можно было проверить на совпадение маршрут);

* И последнее: проверяем, что на странице есть 1 комментарий. В Symfony кое-какие некорректные в CSS селекторы позаимствованы из jQuery. Как раз один из таких селекторов мы используем — ``div:contains()``.

Опять же при помощи CSS-селектора вместо нажатия на текст (``View``), выбираем нужную ссылку:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Проследите, чтобы новый тест выполнился успешно:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Работа с тестовой базой данных
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

Загрузите фикстуры для окружения ``test``:

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

Отправка формы в функциональном тесте
----------------------------------------------------------------------

Хотите пойти дальше и ещё лучше протестировать приложение? Давайте напишем тест, который имитирует отправку форму, добавляя новый комментарий с фотографией к конференции. Непростая задача, не правда? Как бы не так: только посмотрите на используемый для решения код — это не сложнее того, что мы уже делали ранее:

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

Для отправки формы через метод ``submitForm()`` для начала нужно узнать имена элементов с помощью инструментов разработки в браузере или через панель Form в профилировщике Symfony. Обратите внимание на разумное повторное использование изображения-заглушки!

Снова запустите тесты, чтобы убедиться, что они все прошли:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Теперь мы видим один из плюсов использования одинаковой базы данных в окружении разработки и тестирования — полученный результат можно посмотреть в браузере:

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Выгрузка фикстур
-------------------------------

.. index::
    single: Command;doctrine:fixtures:load

Если запустить тесты ещё раз, то они больше не пройдут. А всё потому, что в базе данных уже есть комментарии, и при повторном выполнении тестов они снова добавятся. Вот поэтому проверка количества комментариев не даст положительного результата. Таким образом, перед каждым выполнением тестов нужно очищать базу данных и заново загружать фикстуры:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Автоматизация рабочего процесса с помощью Make-файлов
------------------------------------------------------------------------------------------------

.. index::
    single: Makefile

Помнить и набирать длинную последовательность команд, чтобы выполнить тесты — досадно и неприятно. Как минимум это всё нужно указать в документации, но оставим это на крайний случай. А что, если автоматизировать такие рутинные задачи? Вдобавок это дало своего рода документацию, помогло остальным разработчикам, и просто ускорило разработку.

.. index::
    single: Command;doctrine:fixtures:load

Один из способов автоматизации выполнения команд — воспользоваться ``Makefile``:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

Обратите внимание на флаг ``-n`` в команде Doctrine; это глобальный флаг для Symfony-команд, который выполняет их в обычном (неинтерактивном) режиме.

Теперь, если вам нужно выполнить тесты, то используйте команду ``make tests``:

.. code-block:: bash

    $ make tests

Очистка базы данных после каждого теста
-------------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Чистить базу данных после каждого запуска тестов, конечно, очень хорошо, но иметь действительно независимые друг от друга тесты — гораздо лучше. Крайне нежелательно, чтобы один тест опирался на результаты от предыдущих. Помимо этого, изменение порядка выполнения тестов также не должно влиять на результат. Как мы сейчас убедимся, наши тесты сейчас не соответствуют описанному выше условию.

Переместите тест ``testConferencePage`` после ``testCommentSubmission``:

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

Теперь тесты завершились неудачно.

.. index::
    single: Doctrine;TestBundle

Для очистки базы данных между прогонами тестов установите DoctrineTestBundle:

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Чтобы установить этот бандл потребуется вручную в терминале подтвердить выполнение его рецепта (так как он не поддерживается "официально"):

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

Активируйте обработчик в PHPUnit:

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

Вот и всё, теперь любые изменения данных в тестах будут автоматически отменены после их выполнения.

Тесты снова заработали:

.. code-block:: bash

    $ make tests

Использование настоящего браузера для функциональных тестов
-----------------------------------------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Только что мы рассмотрели использование специального браузера для функциональных тестов, который напрямую вызывает код Symfony. Однако вместо этого можно задействовать настоящий браузер, отправляя реальные HTTP-запросы. Встречайте, Symfony Panther:

.. code-block:: bash

    $ symfony composer req panther --dev

После установки можно написать тесты, в которых используется настоящий браузер Google Chrome, как показано ниже:

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

Переменная окружения ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` содержит URL-адрес локального веб-сервера.

Выполнение функциональных тестов по принципу чёрного ящика с помощью Blackfire
------------------------------------------------------------------------------------------------------------------------------------------

Приложение `Blackfire player <https://blackfire.io/player>`_ — ещё один способ запустить функциональные тесты. Помимо функционального тестирования, этот инструмент также может проводить тестирование производительности.

Для более подробного изучения этой темы посмотрите шаг "Производительность".

.. sidebar:: Двигаемся дальше

    * `Список проверок в Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ для функциональных тестов;

    * `Документация по PHPUnit <https://phpunit.de/documentation.html>`_;

    * `Библиотека Faker <https://github.com/fzaninotto/Faker>`_ для генерации реалистичных данных в фикстурах;

    * `Документация по компоненту CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * `Symfony-библиотека Panther <https://github.com/symfony/panther>`_ для тестирования в браузере и парсинга сайтов в приложениях Symfony;

    * `Документация по Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
