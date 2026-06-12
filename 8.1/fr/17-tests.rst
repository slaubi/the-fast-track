Tester
======

.. index::
    single: PHPUnit

Comme nous commençons à ajouter de plus en plus de fonctionnalités dans l'application, c'est probablement le bon moment pour parler des tests.

*Fun fact* : j'ai trouvé un bogue en écrivant les tests de ce chapitre.

Symfony s'appuie sur PHPUnit pour les tests unitaires. Installons-le :

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Écrire des tests unitaires
---------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` est la première classe pour laquelle nous allons écrire des tests. Générez un test unitaire :

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Tester le SpamChecker est un défi car nous ne voulons certainement pas utiliser l'API d'OpenAI : ce serait lent, coûteux, et les réponses ne seraient même pas déterministes. Nous allons remplacer la *plateforme* par une fausse.

.. index::
    single: Mock

Écrivons un premier test pour le cas où le modèle ne peut pas être joint :

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

La classe ``InMemoryPlatform`` implémente l'interface de la plateforme sans appeler aucune API externe. À partir d'un callable, elle peut simuler n'importe quel comportement, y compris des échecs. Nous l'enveloppons dans un véritable ``Agent`` afin que la logique du ``SpamChecker`` soit testée pour de vrai.

Quand le modèle est indisponible, les commentaires doivent parvenir à un modérateur humain : le score attendu est ``1``.

Lancez les tests pour vérifier qu'ils passent :

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Ajoutons des tests pour les cas qui fonctionnent :

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

Les *data providers* de PHPUnit nous permettent de réutiliser la même logique de test pour plusieurs scénarios.

Écrire des tests fonctionnels pour les contrôleurs
----------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Tester les contrôleurs est un peu différent de tester une classe PHP "ordinaire" car nous voulons les exécuter dans le contexte d'une requête HTTP.

Créez un test fonctionnel pour le contrôleur Conference :

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

Utiliser ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` à la place de ``PHPUnit\Framework\TestCase`` comme classe de base pour nos tests nous fournit une abstraction bien pratique pour les tests fonctionnels.

La variable ``$client`` simule un navigateur. Au lieu de faire des appels HTTP au serveur, il appelle directement l'application Symfony. Cette stratégie présente plusieurs avantages : elle est beaucoup plus rapide que les allers-retours entre le client et le serveur, mais elle permet aussi aux tests d'analyser l'état des services après chaque requête HTTP.

Ce premier test vérifie que la page d'accueil renvoie une réponse HTTP 200.

Des assertions telles que ``assertResponseIsSuccessful`` sont ajoutées à PHPUnit pour faciliter votre travail. Plusieurs assertions de ce type sont définies par Symfony.

.. tip::

    Nous avons utilisé ``/`` pour l'URL au lieu de la générer avec le routeur. C'est volontaire, car tester les URLs telles qu'elles seront déployées fait partie de ce que nous voulons tester. Si vous la modifiez, les tests vont échouer pour vous rappeler que vous devriez probablement rediriger l'ancienne URL vers la nouvelle, pour être gentil avec les moteurs de recherche et les sites web qui renvoient vers le vôtre.

Configurer l'environnement de test
----------------------------------

.. index::
    single: Symfony Environments

Par défaut, les tests PHPUnit sont exécutés dans l'environnement Symfony ``test`` tel qu'il est défini dans le fichier de configuration de PHPUnit :

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

Pour faire fonctionner les tests, nous devons définir la clé secrète ``OPENAI_API_KEY`` pour cet environnement ``test`` :

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Utiliser une base de données de test
-------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Comme nous l'avons déjà vu, la commande Symfony définit automatiquement la variable d'environnement ``DATABASE_URL`` . Quand ``APP_ENV`` vaut ``test``, comme c'est le cas lors de l'exécution de PHPUnit, cela change le nom de la base de données de ``app`` en ``app_test`` pour que les tests utilisent leur propre base de données :

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Cela est très important car nous aurons besoin d'un jeu de données stable pour exécuter nos tests et nous ne voulons certainement pas écraser celui stocké dans la base de développement.

Avant de pouvoir lancer les tests, nous devons "initialiser" la base de données ``test`` (créez la base de données et jouez les migrations) :

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Si vous lancez les tests maintenant, PHPUnit n'interagira plus avec votre base de données de développement. Pour lancer les nouveaux tests uniquement, passez le chemin de leur classe en argument :

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Lorsqu'un test échoue, il peut être utile d'analyser l'objet Response. Accédez-y grâce à ``$client->getResponse()`` et faites un ``echo`` pour voir à quoi il ressemble.

Définir des factories
---------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

Pour pouvoir tester la liste des commentaires, la pagination et la soumission du formulaire, nous devons remplir la base de données avec quelques données. Et pour garder les tests indépendants les uns des autres, chaque test devrait créer exactement le jeu de données dont il a besoin. Les *object factories* sont l'outil parfait pour ce travail.

Installez Zenstruck Foundry :

.. code-block:: terminal

    $ symfony composer req foundry --dev

Générez une factory pour chaque entité dont les tests ont besoin :

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

Une factory décrit comment construire une entité valide : une valeur par défaut est générée pour chaque propriété grâce à la bibliothèque Faker. Créer un objet via une factory le persiste également. Ajustez les valeurs par défaut des conférences pour les rendre plus réalistes :

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

Définir le slug à ``-`` laisse l'entity listener que nous avons écrit en ajoutant les slugs calculer la vraie valeur : une conférence créée avec la ville ``Amsterdam`` et l'année ``2019`` obtient automatiquement le slug ``amsterdam-2019``.

Faites de même pour les commentaires :

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

Notez la valeur par défaut de ``conference`` : quand un commentaire est créé sans conférence explicite, Foundry en crée une à la volée.

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

Le trait ``Factories`` active les factories dans les tests, et ``ResetDatabase`` réinitialise la base de données au début de chaque exécution des tests.

Décrivons ce qu’il se passe dans ce test :

* Le test crée exactement le jeu de données dont il a besoin : deux conférences et un commentaire, via les factories ;

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

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Soumettre un formulaire dans un test fonctionnel
------------------------------------------------

Voulez-vous passer au niveau supérieur ? Essayez d'ajouter un nouveau commentaire avec une photo sur une conférence, à partir d'un test, en simulant une soumission de formulaire. Cela semble ambitieux, n'est-ce pas ? Regardez le code nécessaire : pas plus compliqué que ce que nous avons déjà écrit :

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

Pour soumettre un formulaire via ``submitForm()``, recherchez les noms de champs grâce aux outils de développement du navigateur ou via l'onglet Form du Symfony Profiler. Notez la réutilisation pratique de l'image en construction !

Relancez les tests pour vérifier que tout est bon :

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Si vous voulez vérifier le résultat dans un navigateur, arrêtez le serveur web et relancer le pour l'environnement ``test`` :

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Relancer les tests
------------------

Si vous exécutez les tests une deuxième fois, ils passent toujours : le trait ``ResetDatabase`` réinitialise la base de données au début de chaque exécution, et chaque test crée exactement le jeu de données dont il a besoin. Il n'y a pas d'état partagé ni de restes d'une exécution précédente :

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatiser votre workflow avec un Makefile
-------------------------------------------

.. index::
    single: Makefile

Il est assez pénible d'avoir à se souvenir d'une séquence de commandes pour exécuter les tests. Cela devrait au moins être documenté, même si cette documentation ne devrait être consultée qu'en dernier recours. Et si on automatisait plutôt les opérations récurrentes ? Cela servirait aussi de documentation rapidement accessible aux autres, et rendrait le développement plus facile et plus productif.

L'utilisation d'un ``Makefile`` est une façon d'automatiser les commandes :

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

    Dans une règle Makefile, l'indentation **doit** être une seule tabulation et non des espaces.

Notez l'option ``-n`` sur la commande Doctrine ; c'est une option standard sur les commandes Symfony qui les rend non interactives.

Chaque fois que vous voulez exécuter les tests, utilisez ``make tests`` :

.. code-block:: terminal

    $ make tests

Réinitialiser la base de données après chaque test
-----------------------------------------------------

.. index::
    single: PHPUnit;Performance

Réinitialiser la base de données après chaque test c'est bien, mais avoir des tests vraiment indépendants c'est encore mieux. Nous ne voulons pas qu'un test s'appuie sur les résultats des précédents. Le changement de l'ordre des tests ne devrait pas changer le résultat. Comme nous allons le découvrir maintenant, ce n'est pas le cas pour le moment.

Déplacez le test ``testConferencePage`` après ``testCommentSubmission`` :

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

Les tests échouent maintenant.

.. index::
    single: Doctrine;TestBundle

Pour réinitialiser la base de données entre les tests, installez DoctrineTestBundle :

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

Vous devrez confirmer l'application de la recette (car il ne s'agit pas d'un bundle "officiellement" supporté) :

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

Et voilà. Toute modification apportée pendant les tests est automatiquement annulée à la fin de chaque test.

Les tests devraient passer à nouveau :

.. code-block:: terminal

    $ make tests

Utiliser un vrai navigateur pour les tests fonctionnels
-------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Les tests fonctionnels utilisent un navigateur spécial qui appelle directement la couche Symfony. Mais vous pouvez aussi utiliser un vrai navigateur et la vraie couche HTTP grâce à Symfony Panther :

.. code-block:: terminal

    $ symfony composer req panther --dev

Vous pouvez ensuite écrire des tests qui utilisent un vrai navigateur Google Chrome avec les modifications suivantes :

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

La variable d'environnement ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contient l'URL du serveur web local.

Choisir le bon type de test
---------------------------

.. index::
    single: Command;make:test

Nous avons créé trois type de tests jusqu'à maintenant. Bien que nous n'ayons utilisé le bundle maker que pour générer des tests unitaires, nous aurions tout aussi bien pu l'utiliser pour générer les classes des autres tests :

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

Le bundle maker supporte la génération des types de tests suivants en fonction de la manière dont vous voulez tester votre application :

* ``TestCase``: Tests PHPUnit basiques ;

* ``KernelTestCase`` : Tests basiques ayant accès aux services Symfony ;

* ``WebTestCase`` : Pour exécuter des scénarios à la manière d'un navigateur, mais sans exécution du code JavaScript ;

* ``ApiTestCase`` : Pour jouer des scénarios orientés API ;

* ``PantherTestCase`` : Pour jouer des scénarios e2e, en utilisant un vrai navigateur ou client HTTP et un vrai serveur web.

Exécuter des tests fonctionnels de boîte noire avec Blackfire
---------------------------------------------------------------

Une autre façon d'effectuer des tests fonctionnels est d'utiliser le `lecteur Blackfire`_. En plus de ce que vous pouvez faire avec les tests fonctionnels, il peut également effectuer des tests de performance.

Lisez l'étape :doc:`Performance <29-performance>`" pour en savoir plus.

.. sidebar:: Aller plus loin

    * `Liste des assertions définies par Symfony`_ pour les tests fonctionnels ;

    * `Documentation de PHPUnit`_ ;

    * La `documentation de Foundry`_ ;

    * La `bibliothèque Faker`_ pour générer des données réalistes dans les fixtures ;

    * La `documentation du composant CssSelector`_ ;

    * La bibliothèque `Symfony Panther`_ pour les tests de navigateurs et le parcours de site web dans les applications Symfony ;

    * La `documentation de Make/Makefile`_.

.. _`lecteur Blackfire`: https://blackfire.io/player
.. _`Liste des assertions définies par Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`Documentation de PHPUnit`: https://phpunit.de/documentation.html
.. _`documentation de Foundry`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`bibliothèque Faker`: https://github.com/FakerPHP/Faker
.. _`documentation du composant CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`documentation de Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
