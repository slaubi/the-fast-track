Tester
======

.. index::
    single: PHPUnit

Comme nous commençons à ajouter de plus en plus de fonctionnalités dans l'application, c'est probablement le bon moment pour parler des tests.

*Fun fact* : j'ai trouvé un bogue en écrivant les tests de ce chapitre.

Symfony s'appuie sur PHPUnit pour les tests unitaires. Installons-le :

.. code-block:: bash

    $ symfony composer req phpunit --dev

Écrire des tests unitaires
---------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

``SpamChecker`` est la première classe pour laquelle nous allons écrire des tests. Générez un test unitaire :

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

Tester le SpamChecker est un défi car nous ne voulons certainement pas utiliser l'API Akismet. Nous allons *mocker* l'API.

.. index::
    single: Mock

Écrivons un premier test pour le cas où l'API renverrai une erreur :

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

La classe ``MockHttpClient`` permet de simuler n'importe quel serveur HTTP. Elle prend un tableau d'instances ``MockResponse`` contenant le corps et les en-têtes de réponse attendus.

Ensuite, nous appelons la méthode ``getSpamScore()`` et vérifions qu'une exception est levée via la méthode``expectException()`` de PHPUnit.

Lancez les tests pour vérifier qu'ils passent :

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Ajoutons des tests pour les cas qui fonctionnent :

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

Les *data providers* de PHPUnit nous permettent de réutiliser la même logique de test pour plusieurs scénarios.

Écrire des tests fonctionnels pour les contrôleurs
----------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Tester les contrôleurs est un peu différent de tester une classe PHP "ordinaire" car nous voulons les exécuter dans le contexte d'une requête HTTP.

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

Créez un test fonctionnel pour le contrôleur Conference :

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

Ce premier test vérifie que la page d'accueil renvoie une réponse HTTP 200.

La variable ``$client`` simule un navigateur. Au lieu de faire des appels HTTP au serveur, il appelle directement l'application Symfony. Cette stratégie présente plusieurs avantages : elle est beaucoup plus rapide que les allers-retours entre le client et le serveur, mais elle permet aussi aux tests d'analyser l'état des services après chaque requête HTTP.

Des assertions telles que ``assertResponseIsSuccessful`` sont ajoutées à PHPUnit pour faciliter votre travail. Plusieurs assertions de ce type sont définies par Symfony.

.. tip::

    Nous avons utilisé ``/`` pour l'URL au lieu de la générer avec le routeur. C'est volontaire, car tester les URLs telles qu'elles seront déployées fait partie de ce que nous voulons tester. Si vous la modifiez, les tests vont échouer pour vous rappeler que vous devriez probablement rediriger l'ancienne URL vers la nouvelle, pour être gentil avec les moteurs de recherche et les sites web qui renvoient vers le vôtre.

.. note::

    Nous aurions pu générer le test grâce au *Maker Bundle* :

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

Les tests PHPUnit sont exécutés dans un environnement ``test`` dédié. Nous devons définir la chaîne secrète ``AKISMET_KEY`` de cet environnement :

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Lorsqu'un test échoue, il peut être utile d'analyser l'objet Response. Accédez-y grâce à ``$client->getResponse()`` et faites un ``echo`` pour voir à quoi il ressemble.

Définir des *fixtures* (données de test)
------------------------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Pour pouvoir tester la liste des commentaires, la pagination et la soumission du formulaire, nous devons remplir la base de données avec quelques données. Nous voulons également que les données soient identiques entre les cycles de tests pour qu'ils réussissent. Les fixtures sont exactement ce dont nous avons besoin.

Installez le composant Doctrine Fixtures :

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Un nouveau répertoire ``src/DataFixtures/`` a été créé lors de l'installation, avec une classe d'exemple prête à être personnalisée. Ajoutez deux conférences et un commentaire pour le moment :

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

Lorsque nous chargerons les données de test, toutes les données présentes seront supprimées, y compris celles de l'admin. Pour éviter cela, modifions les fixtures :

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

    Si vous ne vous souvenez pas quel service vous devez utiliser pour une tâche donnée, utilisez le ``debug:autowiring`` avec un mot-clé :

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Charger des données de test
----------------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

Parcourir un site web avec des tests fonctionnels
-------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Comme nous l'avons vu, le client HTTP utilisé dans les tests simule un navigateur, afin que nous puissions parcourir le site comme si nous utilisions un navigateur.

Ajoutez un nouveau test qui clique sur une page de conférence depuis la page d'accueil :

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

Décrivons ce qu’il se passe dans ce test :

* Comme pour le premier test, nous allons sur la page d'accueil ;

* La méthode ``request()`` retourne une instance de ``Crawler`` qui aide à trouver des éléments sur la page (comme des liens, des formulaires, ou tout ce que vous pouvez atteindre avec des sélecteurs CSS ou XPath) ;

* Grâce à un sélecteur CSS, nous testons que nous avons bien deux conférences listées sur la page d'accueil ;

* On clique ensuite sur le lien "View" (comme il n'est pas possible de cliquer sur plus d'un lien à la fois, Symfony choisit automatiquement le premier qu'il trouve) ;

* Nous vérifions le titre de la page, la réponse et le ``<h2>`` de la page pour être sûr d'être sur la bonne page (nous aurions aussi pu vérifier la route correspondante) ;

* Enfin, nous vérifions qu'il y a 1 commentaire sur la page. ``div:contains()`` n'est pas un sélecteur CSS valide, mais Symfony a quelques ajouts intéressants, empruntés à jQuery.

Au lieu de cliquer sur le texte (i.e. ``View``), nous aurions également pu sélectionner le lien grâce à un sélecteur CSS :

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Vérifiez que le nouveau test passe :

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Utiliser une base de données de test
-------------------------------------

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

Chargez les données de test pour l'environnement/la base de données de ``test`` :

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

Soumettre un formulaire dans un test fonctionnel
------------------------------------------------

Voulez-vous passer au niveau supérieur ? Essayez d'ajouter un nouveau commentaire avec une photo sur une conférence, à partir d'un test, en simulant une soumission de formulaire. Cela semble ambitieux, n'est-ce pas ? Regardez le code nécessaire : pas plus compliqué que ce que nous avons déjà écrit :

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

Pour soumettre un formulaire via ``submitForm()``, recherchez les noms de champs grâce aux outils de développement du navigateur ou via l'onglet Form du Symfony Profiler. Notez la réutilisation pratique de l'image en construction !

Relancez les tests pour vérifier que tout est bon :

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Un des avantages d'utiliser la base de données "dev" pour les tests est que vous pouvez vérifier le résultat dans un navigateur :

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Recharger les données de test
------------------------------

.. index::
    single: Command;doctrine:fixtures:load

Si vous effectuez les tests une deuxième fois, ils devraient échouer. Comme il y a maintenant plus de commentaires dans la base de données, l'assertion qui vérifie le nombre de commentaires est erronée. Nous devons réinitialiser l'état de la base de données entre chaque exécution, en rechargeant les données de test avant chacune d'elles :

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatiser votre workflow avec un Makefile
-------------------------------------------

.. index::
    single: Makefile

Il est assez pénible d'avoir à se souvenir d'une séquence de commandes pour exécuter les tests. Cela devrait au moins être documenté, même si cette documentation ne devrait être consultée qu'en dernier recours. Et si on automatisait plutôt les opérations récurrentes ? Cela servirait aussi de documentation rapidement accessible aux autres, et rendrait le développement plus facile et plus productif.

.. index::
    single: Command;doctrine:fixtures:load

L'utilisation d'un ``Makefile`` est une façon d'automatiser les commandes :

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

Notez l'option ``-n`` sur la commande Doctrine ; c'est une option standard sur les commandes Symfony qui les rend non interactives.

Chaque fois que vous voulez exécuter les tests, utilisez ``make tests`` :

.. code-block:: bash

    $ make tests

Réinitialiser la base de données après chaque test
-----------------------------------------------------

.. index::
    single: PHPUnit;Performance

Réinitialiser la base de données après chaque test c'est bien, mais avoir des tests vraiment indépendants c'est encore mieux. Nous ne voulons pas qu'un test s'appuie sur les résultats des précédents. Le changement de l'ordre des tests ne devrait pas changer le résultat. Comme nous allons le découvrir maintenant, ce n'est pas le cas pour le moment.

Déplacez le test ``testConferencePage`` après ``testCommentSubmission`` :

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

Les tests échouent maintenant.

.. index::
    single: Doctrine;TestBundle

Pour réinitialiser la base de données entre les tests, installez DoctrineTestBundle :

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Vous devrez confirmer l'application de la recette (car il ne s'agit pas d'un bundle "officiellement" supporté) :

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

Activez le *listener* de PHPUnit :

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

Et voilà. Toute modification apportée pendant les tests est automatiquement annulée à la fin de chaque test.

Les tests devraient passer à nouveau :

.. code-block:: bash

    $ make tests

Utiliser un vrai navigateur pour les tests fonctionnels
-------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Les tests fonctionnels utilisent un navigateur spécial qui appelle directement la couche Symfony. Mais vous pouvez aussi utiliser un vrai navigateur et la vraie couche HTTP grâce à Symfony Panther :

.. code-block:: bash

    $ symfony composer req panther --dev

Vous pouvez ensuite écrire des tests qui utilisent un vrai navigateur Google Chrome avec les modifications suivantes :

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

La variable d'environnement ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contient l'URL du serveur web local.

Exécuter des tests fonctionnels de boîte noire avec Blackfire
---------------------------------------------------------------

Une autre façon d'effectuer des tests fonctionnels est d'utiliser le `lecteur Blackfire <https://blackfire.io/player>`_. En plus de ce que vous pouvez faire avec les tests fonctionnels, il peut également effectuer des tests de performance.

Reportez-vous à l'étape "Performance" pour en savoir plus.

.. sidebar:: Aller plus loin

    * `Liste des assertions définies par Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ pour les tests fonctionnels ;

    * `Documentation de PHPUnit <https://phpunit.de/documentation.html>`_ ;

    * La `bibliothèque Faker <https://github.com/fzaninotto/Faker>`_ pour générer des données réalistes dans les fixtures ;

    * La `documentation du composant CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_ ;

    * La bibliothèque `Symfony Panther <https://github.com/symfony/panther>`_ pour les tests de navigateurs et le parcours de site web dans les applications Symfony ;

    * La `documentation de Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
