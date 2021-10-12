测试
======

.. index::
    single: PHPUnit

我们开始在程序中加入越来越多的功能，很可能现在是谈谈测试的时候了。

*一件有趣的事*：我在本章中写测试的时候，发现了之前的一个错误。

Symfony 使用 PHPUnit 进行单元测试。我们来安装它：

.. code-block:: bash

    $ symfony composer req phpunit --dev

编写单元测试
------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` 是我们第一个要测试的类。生成一个单元测试：

.. code-block:: bash

    $ symfony console make:test TestCase SpamCheckerTest

测试 SpamChecker 类有点挑战，因为我们当然不想测试的时候去调用 Akismet 的 API。我们会去 *模拟* 这个 API。

.. index::
    single: Mock

我们来测试下 API 返回错误的情况来作为第一个例子：

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

``MockHttpClient`` 类可用来模拟测试任何 HTTP 服务器。它接收一个元素为 ``MockResponse`` 实例的数组，这个数组包含期待的 HTTP 应答头和应答体。

然后，我们调用 ``getSpamScore()`` 方法，再用 PHPUnit 的 ``expectException()`` 方法来检查是否有异常抛出。

运行测试来检查它们是否通过：

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

我们来增加测试用来，来测试代码正常运行的情况：

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

PHPUnit 的 *data providers* 让我们可以在多个测试用例中复用同一个测试逻辑。

为控制器编写功能测试
------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

测试控制器与测试一个“常规”的 PHP 类稍有不同，因为我们要在一个 HTTP 请求上下文中来测试控制器。

为会议的控制器创建一个功能测试：

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

用 ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` 来代替 ``PHPUnit\Framework\TestCase`` 作为测试的基类，这为我们的功能测试提供了一层抽象。

``$client`` 变量模拟一个浏览器。它会直接调用 Symfony 应用，而不是发送 HTTP 请求给 web 服务器。这个策略有一些优点：相比客户端和服务器端之间的来来回回，它更加快速；而且每次 HTTP 请求完成后，测试可以探查到服务的状态。

这第一个测试先检查首页是否返回 200 状态码的 HTTP 应答。

类似 ``assertResponseIsSuccessful`` 这样的断言被包装在 PHPUnit 之上，以便简化你的测试工作。Symfony 定义了很多这样的断言。

.. tip::

    我们硬编码了 ``/`` 这个 URL，而非用路由来生成它。这样做是有意为之的，因为测试用户所使用的URL也是我们测试工作的一部分。如果修改了路由的路径，测试就会失败，这会是个很好的提醒，让你知道或许很有必要把老的 URL 重定向到新的 URL，这样对于搜索引擎和已经链接到你网站的第三方网站会比较友好。

.. note::

    我们其实也可以用 *Maker Bundle* 来生成这个测试：

    .. code-block:: bash

        $ symfony console make:test WebTestCase Controller\\ConferenceController

配置测试环境
------------------

.. index::
    single: Symfony Environments

默认情况下，PHPUnit 的测试运行在 Symfony 的 ``test`` 环境下，这是由  PHPUnit 的配置文件定义的：

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

为了使测试能够运行，我们必须为 ``test`` 环境设置 ``AKISMET_KEY`` 这个秘钥：

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

.. note::

    正如前面有一章里所示，``APP_ENV=test`` 意味着 ``APP_ENV`` 环境变量是在命令的上下文中设置的。在 Windows 系统上要用 ``--env=test``：``symfony console secrets:set AKISMET_KEY --env=test``

使用测试数据库
---------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

正如我们看到的，Symfony 的命令行会自动暴露 ``DATABASE_URL`` 这个环境变量。当 ``APP_ENV`` 的值是 ``test`` 时，比如在运行 PHPUnit 时设置的那样，它会把数据库的名字从 ``main`` 改为 ``main_test``，这样的话测试会使用它们专门的数据库。这是很重要的，因为我们需要有稳定的数据来进行测试，而且我们当然也不希望测试会改写我们在开发环境数据库里存储的内容。

在可以运行测试之前，我们需要“初始化” ``test`` 数据库（创建这个数据库并对它迁移）：

.. code-block:: bash

    $ APP_ENV=test symfony console doctrine:database:create
    $ APP_ENV=test symfony console doctrine:migrations:migrate -n

如果你现在运行测试，PHPUnit 不会再影响到开发环境里的数据库。如果只是运行新的测试，就要传入这些类的路径：

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

请注意，即便在运行 PHPUnit 时，我们也明确设置了 ``APP_ENV``，这样 Symfony 命令行才能把数据库名设置为 ``main_test``。

.. tip::

    当一个测试失败时，探查应答对象会很有用。用 ``echo`` 来输出 ``$client->getResponse()``，看看它里面有什么。

定义 Fixtures 数据
----------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

我们需要用一些数据来填充数据库，这样才能测试评论列表、分页和表单提交。我们需要在不同的测试间确保数据是一样的。Fixtures 就是我们需要的。

安装 Doctrine Fixtures 这个 bundle：

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

安装的过程会创建新目录 ``src/DataFixtures/``，里面有一个样例类，你可以去定制它。现在先加上 2 个会议和 1 条评论：

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

当我们载入 fixture 数据时，所有之前的数据都会被移除，包括管理员账户的数据。为了避免这种情况，让我们来把管理员账户加进 fixture 里：

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

    如果你不确定要为某个任务使用哪个服务，可以用 ``debug:autowiring``，再加上一些关键词：

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

载入 fixture 数据
---------------------

.. index:: ! Command;doctrine:fixtures:load

为 ``test`` 环境对应的数据库载入 fixture 数据：

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load

在功能测试中爬取网站
------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

如我们看到的那样，测试中用到的 HTTP 客户端会模拟浏览器，所以我们可以用它来浏览网站，就好像用一个无头浏览器一样。

新建一个测试，用它在首页上点击一个会议页面的链接。

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

让我们用大白话来描述下这个测试做了什么：

* 就像第一个测试那样，我们来到首页；

* ``request()`` 方法返回一个 ``Crawler`` 实例，该实例可以用来找到页面中的元素（比如链接、表单或任何你可以用 CSS 选择器或 XPath 找到的元素）。

* 借助于 CSS 选择器，我们断言首页上列出了 2 个会议；

* 然后我们点击 "View" 链接（Symfony 不能同时点击多于一个链接，所以它会自动选择找到的第一个链接）；

* 我们验证了页面标题，返回的应答和页面的 ``<h2>`` 标签，以确保我们是在正确的页面上（我们也可以验证匹配的路由）；

* 最后，我们验证页面上有一条评论。``div:contains()`` 并不是合规的 CSS 选择器，但 Symfony 借鉴了 jQuery，对 CSS 选择器做了一些增强。

我们也可以用 CSS 选择器来选择一个链接，而不是通过点击链接文本（也就是本例中的 ``View``）：

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

检查新的测试能否通过：

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

在功能测试中提交表单
------------------------------

你想把水平再提高一个台阶吗？试试看在测试中模拟一次表单提交，提交的数据是一条评论和一个会议的照片。这看上去很有雄心壮志，不是吗？看一下所需的代码，它并不比我们已经写过的更复杂：

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

先用浏览器的开发者工具或者 Symfony 分析器里的 Form 面板来找到 input 元素的名字，然后才能用 ``submitForm()`` 方法来提交表单。注意我们聪明地重用了“在建中”图片！

再运行下测试，检查测试是否通过：

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

如果你想要在浏览器中检查结果，停止 Web 服务器，并且在 ``test`` 环境下重新运行它：

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

重新载入 fixture 数据
---------------------------

.. index::
    single: Command;doctrine:fixtures:load

如果你再运行一次测试，它应该会失败。由于现在数据库里有更多的评论，检查评论数量的断言就不成立了。我们需要在多次测试之间重置数据库的状态，这是通过在每次测试之前重新载入 fixture 数据来实现的。

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load
    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

用 Makefile 来自动化你的工作流
----------------------------------------

.. index::
    single: Makefile

被迫记住一组命令才能运行测试很烦人。至少这些应该要记录在文档里。但文档应该是最后才考虑的方案。把日常的工作自动化，而不是用文档，你觉得如何？它能实现文档的目的，帮助其他开发者了解项目，也能让他们的生活更轻松，完成工作更迅速。

.. index::
    single: Command;doctrine:fixtures:load

使用 ``Makefile`` 就是自动化执行一组命令的方法之一：

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

    在一个 Makefile 规则里，缩进 **必须** 由单独一个 tab 组成，而不是由空格组成。

注意 Doctrine 命令里的 ``-n`` 选项，它是 Symfony 命令的一个全局选项，让命令采用非交互模式运行。

无论何时你要进行测试，就使用 ``make tests``：

.. code-block:: bash

    $ make tests

在每次测试后重置数据库
---------------------------------

.. index::
    single: PHPUnit;Performance

每次测试后重置数据库，这样很好，但是让测试真正独立运行，那就更好了。我们不希望一个测试依赖于前面测试的结果。改变测试的顺序不应该改变结果。现在我们会看到，目前这一点还没有做到。

把 ``testConferencePage`` 测试移动到 ``testCommentSubmission`` 测试后面：

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

现在测试不能通过了。

.. index::
    single: Doctrine;TestBundle

为了在测试之间重置数据库，我们来安装 DoctrineTestBundle：

.. code-block:: bash
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: bash

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

你需要确认 recipe 的执行（因为它不是一个 *官方* 支持的 bundle）：

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

启用 PHPUnit 的监听器：

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

完成了。对数据库所做的任何改变在测试结束时都会自动回滚。

测试应该再次通过了：

.. code-block:: bash

    $ make tests

用真正的浏览器来进行功能测试
------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

功能测试用一个特殊的浏览器来直接调用 Symfony 层。但你也可以借助 Symfony Panther，用一个真正的浏览器和真正的 HTTP 层。

.. code-block:: bash

    $ symfony composer req panther --dev

进行如下改动，你就可以让测试调用真正的谷歌 Chrome 浏览器：

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

``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` 环境变量包含了本地 web 服务器的 URL。

用 Blackfire 进行黑盒功能测试
--------------------------------------

另一个运行功能测试的方法，就是使用 `Blackfire 播放器 <https://blackfire.io/player>`_。你除了能运行功能测试以外，还可以运行性能测试。

参考关于“性能”的步骤来了解更多这方面的内容。

.. sidebar:: 深入学习

    * `Symfony 定义的断言列表 <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_，在功能测试里会用到；

    * `PHPUnit 文档 <https://phpunit.de/documentation.html>`_；

    * 用来生成仿真 fixture 数据的 `Faker 库 <https://github.com/FakerPHP/Faker>`_；

    * `CssSelector 组件文档 <https://symfony.com/doc/current/components/css_selector.html>`_；

    * `Symfony Panther <https://github.com/symfony/panther>`_ 库，用于在浏览器中对 Symfony 应用进行测试，也用来对应用抓取网页；

    * `Make/Makefile文档 <https://www.gnu.org/software/make/manual/make.html>`_。
