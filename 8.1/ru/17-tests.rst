Тестирование
========================

.. index::
    single: PHPUnit

Поскольку мы всё больше и больше добавляем новой функциональности в приложение, наверное, сейчас самое время обсудить тестирование.

*Забавный факт*: я нашёл баг во время написания тестов в этой главе.

Symfony использует PHPUnit для модульного тестирования. Давайте установим его:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Написание модульных тестов
--------------------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

Давайте начнём с тестирования класса ``SpamChecker``. Создайте образец теста для него:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Протестировать SpamChecker непросто, потому что нам явно не стоит обращаться к API OpenAI во время тестирования: это было бы медленно, дорого, а ответы даже не были бы детерминированными. Поэтому заменим *платформу* фиктивной.

.. index::
    single: Mock

Давайте напишем наш первый тест для случая, когда API возвращает ошибку:

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

Класс ``InMemoryPlatform`` реализует интерфейс платформы, не обращаясь ни к какому внешнему API. Получив callable, он может имитировать любое поведение, включая сбои. Мы оборачиваем его в настоящий ``Agent``, чтобы логика ``SpamChecker`` тестировалась по-настоящему.

Когда модель недоступна, комментарии должны попадать к модератору-человеку: ожидаемая оценка — ``1``.

Запустите тесты и посмотрите, что все они успешно выполнились:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Давайте добавим тесты на позитивный сценарий:

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

В PHPUnit есть провайдеры данных, при помощи которых в разных тестах можно использовать одни и те же тестовые данные.

Написание функциональных тестов для контроллеров
--------------------------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Тестирование контроллеров несколько отличается от тестирования "обычного" PHP-класса, поскольку процесс происходит в контексте HTTP-запроса.

Создайте функциональный тест для контроллера конференций:

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

Используя  ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase``, как базовый класс вместо ``PHPUnit\Framework\TestCase`` позволяет нам добится хорошей абстракции для функциональных тестов.

Переменная ``$client`` предназначена для имитации браузера. Вместо отправки HTTP-запросов на сервер, происходит обращение к приложению Symfony напрямую. Такой подход имеет свои преимущества: он намного быстрее реального взаимодействия между клиентом и сервером, а ещё позволяет контролировать состояние сервисов после каждого HTTP-запроса.

Первый тест проверяет, что главная страница возвращает статус 200 в HTTP-ответе.

Такие проверки, как ``assertResponseIsSuccessful``, добавлены поверх PHPUnit, чтобы упростить вам работу. В Symfony ещё много подобных проверок.

.. tip::

    Мы использовали ``/`` в качестве URL-адреса, вместо получения его через маршрутизатор. Это сделано не случайно, так как тестирование URL-адресов, используемых пользователем,  являются частью того, что мы хотим протестировать. Если вы поменяете адрес, тесты уже не будут успешно выполняться. Это послужит полезным напоминанием о том, что вам, скорее всего, нужно сделать редирект со старого адреса на новый, чтобы поисковые машины и ссылающиеся сайты узнали об этом.

Настройка тестового окружения
--------------------------------------------------------

.. index::
    single: Symfony Environments

По умолчанию тесты PHPUnit запускаются в окружении Symfony под названием ``test``. Параметры заданы в конфигурационном файле PHPUnit:

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

Для того, чтобы заработали тесты, установим переменную окружения с секретным ключом ``OPENAI_API_KEY`` в окружении ``test``:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Работа с тестовой базой данных
--------------------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Как показано ранее, CLI-команды Symfony автоматически подставляют переменную окружения ``DATABASE_URL``. Когда значение ``APP_ENV`` равно ``test`` (например, как при выполнении тестов PHPUnit), Symfony изменит имя базы данных с ``app`` на ``app_test``, создав отдельную базу данных для тестов.

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Это очень важно, поскольку нам понадобятся стабильные данные для запуска наших тестов, и мы, конечно, не хотим переопределять то, что мы сохранили в базе данных разработки.

Для запуска тестов сначала нужно "инициализировать" базу данных ``test`` (т.е. создать новую базу данных и выполнить миграцию):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Если мы теперь запустим тесты, PHPUnit больше не будет взаимодействовать с нашей рабочей базой данной. Для выполнения только новых тестов передайте путь к нужному тестируемому классу в CLI-команде:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Если тест не проходит, имеет смысл посмотреть на состояние объекта Response — вывести на экран результат выполнения метода ``$client->getResponse()`` через конструкцию ``echo``.

Определение фабрик
-------------------------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

Для тестирования списка комментариев, пагинации и отправки формы нужно заполнить базу данных какими-нибудь данными. А чтобы тесты оставались независимыми друг от друга, каждый тест должен создавать ровно тот набор данных, который ему нужен. *Объектные фабрики* — идеальный инструмент для этой работы.

Установите Zenstruck Foundry:

.. code-block:: terminal

    $ symfony composer req foundry --dev

Сгенерируйте фабрику для каждой сущности, которая нужна тестам:

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

Фабрика описывает, как построить корректную сущность: для каждого свойства генерируется значение по умолчанию благодаря библиотеке Faker. Создание объекта через фабрику также сохраняет его. Настройте значения по умолчанию для конференций, чтобы они были более реалистичными:

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

Если задать слагу значение ``-``, то прослушиватель сущностей, написанный нами при добавлении слагов, вычислит настоящее значение: конференция, созданная с городом ``Amsterdam`` и годом ``2019``, автоматически получит слаг ``amsterdam-2019``.

Сделайте то же самое для комментариев:

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

Обратите внимание на значение по умолчанию для ``conference``: когда комментарий создаётся без явно указанной конференции, Foundry создаёт её на лету.

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

Трейт ``Factories`` включает фабрики в тестах, а ``ResetDatabase`` сбрасывает базу данных в начале каждого запуска тестов.

Давайте опишем, какие действия должны происходить в тесте  на простом английском языке:

* Тест создаёт ровно тот набор данных, который ему нужен: две конференции и один комментарий, через фабрики;

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

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Отправка формы в функциональном тесте
----------------------------------------------------------------------

Хотите пойти дальше и ещё лучше протестировать приложение? Давайте напишем тест, который имитирует отправку форму, добавляя новый комментарий с фотографией к конференции. Непростая задача, не правда? Как бы не так: только посмотрите на используемый для решения код — это не сложнее того, что мы уже делали ранее:

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

Для отправки формы через метод ``submitForm()`` для начала нужно узнать имена элементов с помощью инструментов разработки в браузере или через панель Form в профилировщике Symfony. Обратите внимание на разумное повторное использование изображения-заглушки!

Снова запустите тесты, чтобы убедиться, что они все прошли:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Если вы хотите посмотреть результат в браузере, остановите веб-сервер и перезапустите его в окружении ``test``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Повторный запуск тестов
-------------------------------

Если запустить тесты ещё раз, они по-прежнему проходят: трейт ``ResetDatabase`` сбрасывает базу данных в начале каждого запуска, а каждый тест создаёт ровно тот набор данных, который ему нужен. Нет ни общего состояния, ни остатков от предыдущего запуска:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Автоматизация рабочего процесса с помощью Make-файлов
------------------------------------------------------------------------------------------------

.. index::
    single: Makefile

Помнить и набирать длинную последовательность команд, чтобы выполнить тесты — досадно и неприятно. Как минимум это всё нужно указать в документации, но оставим это на крайний случай. А что, если автоматизировать такие рутинные задачи? Вдобавок это дало своего рода документацию, помогло остальным разработчикам, и просто ускорило разработку.

Один из способов автоматизации выполнения команд — воспользоваться ``Makefile``:

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

    Следуя правилам Makefile отступ **должен** состоять из одного символа табуляции вместо пробелов.

Обратите внимание на флаг ``-n`` в команде Doctrine; это глобальный флаг для Symfony-команд, который выполняет их в обычном (неинтерактивном) режиме.

Теперь, если вам нужно выполнить тесты, то используйте команду ``make tests``:

.. code-block:: terminal

    $ make tests

Очистка базы данных после каждого теста
-------------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Чистить базу данных после каждого запуска тестов, конечно, очень хорошо, но иметь действительно независимые друг от друга тесты — гораздо лучше. Крайне нежелательно, чтобы один тест опирался на результаты от предыдущих. Помимо этого, изменение порядка выполнения тестов также не должно влиять на результат. Как мы сейчас убедимся, наши тесты сейчас не соответствуют описанному выше условию.

Переместите тест ``testConferencePage`` после ``testCommentSubmission``:

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

Теперь тесты завершились неудачно.

.. index::
    single: Doctrine;TestBundle

Для очистки базы данных между прогонами тестов установите DoctrineTestBundle:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

Чтобы установить этот бандл потребуется вручную в терминале подтвердить выполнение его рецепта (так как он не поддерживается "официально"):

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

Вот и всё, теперь любые изменения данных в тестах будут автоматически отменены после их выполнения.

Тесты снова заработали:

.. code-block:: terminal

    $ make tests

Использование настоящего браузера для функциональных тестов
-----------------------------------------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Только что мы рассмотрели использование специального браузера для функциональных тестов, который напрямую вызывает код Symfony. Однако вместо этого можно задействовать настоящий браузер, отправляя реальные HTTP-запросы. Встречайте, Symfony Panther:

.. code-block:: terminal

    $ symfony composer req panther --dev

После установки можно написать тесты, в которых используется настоящий браузер Google Chrome, как показано ниже:

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

Переменная окружения ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` содержит URL-адрес локального веб-сервера.

Выбор правильного типа теста
-----------------------------------------------------

.. index::
    single: Command;make:test

На данный момент мы создали три различных типа тестов. Хотя мы использовали пакет maker только для генерации класса модульного теста, мы могли бы использовать его и для генерации других тестовых классов:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

Пакет maker поддерживает генерацию следующих типов тестов в зависимости от того, как вы хотите протестировать своё приложение:

* ``TestCase``: обычные тесты PHPUnit;

* ``KernelTestCase``: базовые тесты, имеющие доступ к сервисам Symfony;

* ``WebTestCase``: для тестов, предполагающих для работы наличие браузера, но без выполнения JavaScript кода;

* ``ApiTestCase``: для тестов API;

* ``PantherTestCase``: для запуска сценариев e2e с использованием реального браузера или HTTP-клиента и реального веб-сервера.

Выполнение функциональных тестов по принципу чёрного ящика с помощью Blackfire
------------------------------------------------------------------------------------------------------------------------------------------

Приложение `Blackfire player`_ — ещё один способ запустить функциональные тесты. Помимо функционального тестирования, этот инструмент также может проводить тестирование производительности.

Для более подробного изучения этой темы посмотрите шаг :doc:`Производительность <29-performance>`.

.. sidebar:: Двигаемся дальше

    * `Список проверок в Symfony`_ для функциональных тестов;

    * `Документация по PHPUnit`_;

    * `Документация Foundry`_;

    * `Библиотека Faker`_ для генерации реалистичных данных в фикстурах;

    * `Документация по компоненту CssSelector`_;

    * `Symfony-библиотека Panther`_ для тестирования в браузере и парсинга сайтов в приложениях Symfony;

    * `Документация по Make/Makefile`_.

.. _`Blackfire player`: https://blackfire.io/player
.. _`Список проверок в Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`Документация по PHPUnit`: https://phpunit.de/documentation.html
.. _`Документация Foundry`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`Библиотека Faker`: https://github.com/FakerPHP/Faker
.. _`Документация по компоненту CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony-библиотека Panther`: https://github.com/symfony/panther
.. _`Документация по Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
