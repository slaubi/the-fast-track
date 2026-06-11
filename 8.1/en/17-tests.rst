Testing
=======

.. index::
    single: PHPUnit

As we start to add more and more functionality to the application, it is probably the right time to talk about testing.

*Fun fact*: I found a bug while writing the tests in this chapter.

Symfony relies on PHPUnit for unit tests. Let's install it:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Writing Unit Tests
------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` is the first class we are going to write tests for. Generate a unit test:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Testing the SpamChecker is a challenge as we certainly don't want to hit the OpenAI API: it would be slow, expensive, and the answers would not even be deterministic. We are going to replace the *platform* with a fake one.

.. index::
    single: Mock

Let's write a first test for when the model cannot be reached:

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

The ``InMemoryPlatform`` class implements the platform interface without calling any external API. Given a callable, it can simulate any behavior, including failures. We wrap it in a real ``Agent`` so that the ``SpamChecker`` logic is tested for real.

When the model is down, comments must reach a human moderator: the expected score is ``1``.

Run the tests to check that they pass:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Let's add tests for the happy path:

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

PHPUnit data providers allow us to reuse the same test logic for several test cases.

Writing Functional Tests for Controllers
----------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Testing controllers is a bit different than testing a "regular" PHP class as we want to run them in the context of an HTTP request.

Create a functional test for the Conference controller:

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

Using ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` instead of ``PHPUnit\Framework\TestCase`` as a base class for our tests gives us a nice abstraction for functional tests.

The ``$client`` variable simulates a browser. Instead of making HTTP calls to the server though, it calls the Symfony application directly. This strategy has several benefits: it is much faster than having round-trips between the client and the server, but it also allows the tests to introspect the state of the services after each HTTP request.

This first test checks that the homepage returns a 200 HTTP response.

Assertions such as ``assertResponseIsSuccessful`` are added on top of PHPUnit to ease your work. There are many such assertions defined by Symfony.

.. tip::

    We have used ``/`` for the URL instead of generating it via the router. This is done on purpose as testing end-user URLs is part of what we want to test. If you change the route path, tests will break as a nice reminder that you should probably redirect the old URL to the new one to be nice with search engines and websites that link back to your website.

Configuring the Test Environment
--------------------------------

.. index::
    single: Symfony Environments

By default, PHPUnit tests are run in the ``test`` Symfony environment as defined in the PHPUnit configuration file:

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

To make tests work, we must set the ``OPENAI_API_KEY`` secret for this ``test`` environment:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Working with a Test Database
----------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

As we have already seen, the Symfony CLI automatically exposes the ``DATABASE_URL`` environment variable. When ``APP_ENV`` is ``test``, like set when running PHPUnit, the database name changes from ``app`` to ``app_test`` so that tests have their very own database:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

This is very important as we will need some stable data to run our tests and we certainly don't want to override what we stored in the development database.

Before being able to run the test, we need to "initialize" the ``test`` database (create the database and migrate it):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

If you now run the tests, PHPUnit won't interact with your development database anymore. To only run the new tests, pass the path to their class path:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    When a test fails, it might be useful to introspect the Response object. Access it via ``$client->getResponse()`` and ``echo`` it to see what it looks like.

Defining Fixtures
-----------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

To be able to test the comment list, the pagination, and the form submission, we need to populate the database with some data. And we want the data to be the same between test runs to make the tests pass. Fixtures are exactly what we need.

Install the Doctrine Fixtures bundle:

.. code-block:: terminal

    $ symfony composer req orm-fixtures --dev

A new ``src/DataFixtures/`` directory has been created during the installation with a sample class, ready to be customized. Add two conferences and one comment for now:

.. code-block:: diff
    :caption: patch_file

    --- i/src/DataFixtures/AppFixtures.php
    +++ w/src/DataFixtures/AppFixtures.php
    @@ -2,6 +2,8 @@

     namespace App\DataFixtures;

    +use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\FixturesBundle\Fixture;
     use Doctrine\Persistence\ObjectManager;

    @@ -9,8 +11,24 @@ class AppFixtures extends Fixture
     {
         public function load(ObjectManager $manager): void
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

When we load the fixtures, all data is removed; including the admin user. To avoid that, let's add the admin user in the fixtures:

.. code-block:: diff

    --- i/src/DataFixtures/AppFixtures.php
    +++ w/src/DataFixtures/AppFixtures.php
    @@ -2,13 +2,20 @@

     namespace App\DataFixtures;

    +use App\Entity\Admin;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\FixturesBundle\Fixture;
     use Doctrine\Persistence\ObjectManager;
    +use Symfony\Component\PasswordHasher\Hasher\PasswordHasherFactoryInterface;

     class AppFixtures extends Fixture
     {
    +    public function __construct(
    +        private PasswordHasherFactoryInterface $passwordHasherFactory,
    +    ) {
    +    }
    +
         public function load(ObjectManager $manager): void
         {
             $amsterdam = new Conference();
    @@ -30,6 +37,12 @@ class AppFixtures extends Fixture
             $comment1->setText('This was a great conference.');
             $manager->persist($comment1);

    +        $admin = new Admin();
    +        $admin->setRoles(['ROLE_ADMIN']);
    +        $admin->setUsername('admin');
    +        $admin->setPassword($this->passwordHasherFactory->getPasswordHasher(Admin::class)->hash('admin'));
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

    If you don't remember which service you need to use for a given task, use the ``debug:autowiring`` with some keyword:

    .. code-block:: terminal

        $ symfony console debug:autowiring hasher

Loading Fixtures
----------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures for the ``test`` environment/database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:fixtures:load --env=test

Crawling a Website in Functional Tests
--------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

As we have seen, the HTTP client used in the tests simulates a browser, so we can navigate through the website as if we were using a headless browser.

Add a new test that clicks on a conference page from the homepage:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -14,4 +14,19 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }
    +
    +    public function testConferencePage(): void
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

Let's describe what happens in this test in plain English:

* Like the first test, we go to the homepage;

* The ``request()`` method returns a ``Crawler`` instance that helps find elements on the page (like links, forms, or anything you can reach with CSS selectors or XPath);

* Thanks to a CSS selector, we assert that we have two conferences listed on the homepage;

* We then click on the "View" link (as it cannot click on more than one link at a time, Symfony automatically chooses the first one it finds);

* We assert the page title, the response, and the page ``<h2>`` to be sure we are on the right page (we could also have checked for the route that matches);

* Finally, we assert that there is 1 comment on the page. ``div:contains()`` is not a valid CSS selector, but Symfony has some nice additions, borrowed from jQuery.

Instead of clicking on text (i.e. ``View``), we could have selected the link via a CSS selector as well:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Check that the new test is green:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Submitting a Form in a Functional Test
--------------------------------------

Do you want to get to the next level? Try adding a new comment with a photo on a conference from a test by simulating a form submission. That seems ambitious, doesn't it? Look at the needed code: not more complex than what we already wrote:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -29,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
             $this->assertSelectorExists('div:contains("There are 1 comments")');
         }
    +
    +    public function testCommentSubmission(): void
    +    {
    +        $client = static::createClient();
    +        $client->request('GET', '/conference/amsterdam-2019');
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

To submit a form via ``submitForm()``, find the input names thanks to the browser DevTools or via the Symfony Profiler Form panel. Note the clever re-use of the under construction image!

Run the tests again to check that everything is green:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

If you want to check the result in a browser, stop the Web server and re-run it for the ``test`` environment:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Reloading the Fixtures
----------------------

.. index::
    single: Command;doctrine:fixtures:load

If you run the tests a second time, they should fail. As there are now more comments in the database, the assertion that checks the number of comments is broken. We need to reset the state of the database between each run by reloading the fixtures before each run:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:fixtures:load --env=test
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automating your Workflow with a Makefile
----------------------------------------

.. index::
    single: Makefile

Having to remember a sequence of commands to run the tests is annoying. This should at least be documented. But documentation should be a last resort. Instead, what about automating day to day activities? That would serve as documentation, help discovery by other developers, and make developer lives easier and fast.

.. index::
    single: Command;doctrine:fixtures:load

Using a ``Makefile`` is one way to automate commands:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:database:drop --force --env=test || true
    	symfony console doctrine:database:create --env=test
    	symfony console doctrine:migrations:migrate -n --env=test
    	symfony console doctrine:fixtures:load -n --env=test
    	symfony php bin/phpunit $(MAKECMDGOALS)
    .PHONY: tests

.. warning::

    In a Makefile rule, indentation **must** consist of a single tab character instead of spaces.

Note the ``-n`` flag on the Doctrine command; it is a global flag on Symfony commands that makes them non interactive.

Whenever you want to run the tests, use ``make tests``:

.. code-block:: terminal

    $ make tests

Resetting the Database after each Test
--------------------------------------

.. index::
    single: PHPUnit;Performance

Resetting the database after each test run is nice, but having truly independent tests is even better. We don't want one test to rely on the results of the previous ones. Changing the order of the tests should not change the outcome. As we're going to figure out now, this is not the case for the moment.

Move the ``testConferencePage`` test after the ``testCommentSubmission`` one:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -15,21 +15,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }

    -    public function testConferencePage(): void
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
         public function testCommentSubmission(): void
         {
             $client = static::createClient();
    @@ -44,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }
    +
    +    public function testConferencePage(): void
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

Tests now fail.

.. index::
    single: Doctrine;TestBundle

To reset the database between tests, install DoctrineTestBundle:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

You will need to confirm the execution of the recipe (as it is not an "officially" supported bundle):

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

And done. Any changes done in tests are now automatically rolled-back at the end of each test.

Tests should be green again:

.. code-block:: terminal

    $ make tests

Using a real Browser for Functional Tests
-----------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Functional tests use a special browser that calls the Symfony layer directly. But you can also use a real browser and the real HTTP layer thanks to Symfony Panther:

.. code-block:: terminal

    $ symfony composer req panther --dev

You can then write tests that use a real Google Chrome browser with the following changes:

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

The ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` environment variable contains the URL of the local web server.

Choosing the Right Test Type
----------------------------

.. index::
    single: Command;make:test

We have created three different types of tests so far. While we have only used the maker bundle to generate the unit test class, we could have used it to generate the other test classes as well:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

The maker bundle supports generating the following types of tests depending on how you want to test your application:

* ``TestCase``: Basic PHPUnit tests;

* ``KernelTestCase``: Basic tests that have access to Symfony services;

* ``WebTestCase``: To run browser-like scenarios, but that don't execute JavaScript code;

* ``ApiTestCase``: To run API-oriented scenarios;

* ``PantherTestCase``: To run e2e scenarios, using a real-browser or HTTP client and a real web server.

Running Black Box Functional Tests with Blackfire
-------------------------------------------------

Another way to run functional tests is to use the `Blackfire player`_. In addition to what you can do with functional tests, it can also perform performance tests.

Read the :doc:`Performance <29-performance>` step to learn more.

.. sidebar:: Going Further

    * `List of assertions defined by Symfony`_ for functional tests;

    * `PHPUnit docs`_;

    * The `Faker library`_ to generate realistic fixtures data;

    * The `CssSelector component docs`_;

    * The `Symfony Panther`_ library for browser testing and web crawling in Symfony applications;

    * The `Make/Makefile docs`_.

.. _`Blackfire player`: https://blackfire.io/player
.. _`List of assertions defined by Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`PHPUnit docs`: https://phpunit.de/documentation.html
.. _`Faker library`: https://github.com/FakerPHP/Faker
.. _`CssSelector component docs`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`Make/Makefile docs`: https://www.gnu.org/software/make/manual/make.html
