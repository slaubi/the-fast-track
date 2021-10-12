Test
====

.. index::
    single: PHPUnit

Quando iniziamo ad aggiungere sempre più funzionalità all'applicazione, è probabilmente il momento giusto per parlare di test.

*Curiosità*: ho trovato un bug mentre scrivevo i test in questo capitolo.

Symfony si basa su PHPUnit per i test unitari. Installiamolo:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Scrittura di test unitari
-------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

``SpamChecker`` è la prima classe per cui scriveremo i test. Creiamo un test unitario:

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

Testare lo SpamChecker è una sfida perché di sicuro non vogliamo chiamare l'API di Akismet. Creiamo quindi un mock.

.. index::
    single: Mock

Scriviamo un primo test per lo scenario in cui l'API restituisce un errore:

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

La classe ``MockHttpClient`` permette di simulare qualsiasi server HTTP. Richiede un array di istanze ``MockResponse``, che contengono il body e gli header di risposta attesi.

Poi, chiamiamo il metodo ``getSpamScore()`` e verifichiamo che sia lanciata un'eccezione tramite il metodo ``expectException()`` di PHPUnit.

Eseguiamo i test per verificare che passino:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Aggiungiamo i test per lo scenario positivo:

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

I data provider di PHPUnit ci permettono di riutilizzare la stessa logica di test per diversi casi.

Scrivere test funzionali per i controller
-----------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Testare i controller è un po' diverso dal testare una classe PHP "normale", poiché vogliamo eseguirli nel contesto di una richiesta HTTP.

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

Creiamo un test funzionale per il controller delle conferenze:

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

Questo primo test controlla che la homepage restituisca una risposta HTTP con codice 200.

La variabile ``$client`` simula un browser. Tuttavia, invece di fare chiamate HTTP al server, chiama direttamente l'applicazione Symfony. Questa strategia ha diversi vantaggi: è molto più veloce che avere chiamate in uscita e ritorno tra il client e il server, ma permette anche ai test di effettuare una introspezione sullo stato dei servizi dopo ogni richiesta HTTP.

Per semplificare il lavoro, metodi di asserzione come ``assertResposeIsSuccessful`` sono stati aggiunti a PHPUnit. Symfony definisce molte altre asserzioni.

.. tip::

    Abbiamo usato ``/`` per l'URL invece di generarlo tramite il router. È stato fatto volutamente perché la verifica degli URL finali fa parte di ciò che vogliamo testare. Se si cambia il percorso, i test si romperanno a ricordare che probabilmente bisognerebbe reindirizzare il vecchio URL al nuovo URL, per essere gentili con i motori di ricerca e i siti web che rimandano al vostro.

.. note::

    Avremmo potuto generare il test tramite il maker bundle:

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

I test di PHPUnit vengono eseguiti in un ambiente di ``test``  dedicato. Dobbiamo impostare il secret ``AKISMET_KEY`` per questo ambiente:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Quando un test fallisce, potrebbe essere utile analizzare l'oggetto Response: possiamo ottenerlo tramite ``$client->getResponse()`` e usare ``echo`` per vedere come è fatto.

Definizione delle fixture
-------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Per poter testare la lista dei commenti, la paginazione e l'invio del form, abbiamo bisogno di popolare il database con alcuni dati. E vogliamo che i dati siano gli stessi tra un test e l'altro per far passare i test. Le fixture sono esattamente quello che ci serve.

Installare il bundle Doctrine Fixtures:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Durante l'installazione è stata creata una nuova cartella chiamata ``src/DataFixtures/`` con una classe di esempio, pronta per essere personalizzata. Per il momento aggiungiamo due conferenze e un commento:

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

Quando caricheremo le fixture, tutti i dati saranno rimossi, incluso l'utente admin. Per evitarlo, aggiungiamo l'utente admin nelle fixture:

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

    Se non si ricorda quale servizio sia necessario utilizzare per un determinato scopo, possiamo usare ``debug:autowiring`` con una parola chiave:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Caricamento delle fixture
-------------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

Eseguire il crawling di un sito web nei test funzionali
-------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Come abbiamo visto, il client HTTP utilizzato nei test simula un browser, così possiamo navigare attraverso il sito web come se stessimo usando un browser headless.

Aggiungere un nuovo test che clicca su una pagina della conferenza dalla homepage:

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

Descriviamo in un linguaggio naturale cosa succede in questo test:

* Come il primo test, andiamo alla homepage;

* Il metodo ``request()`` restituisce un'istanza di ``Crawler`` che aiuta a trovare elementi della pagina (come link, form, o qualsiasi cosa si possa raggiungere con i selettori CSS o XPath);

* Grazie ad un selettore CSS, affermiamo di avere due conferenze elencate in homepage;

* Clicchiamo poi  sul link "View" (non potendo cliccare su più di un link alla volta, Symfony sceglie automaticamente il primo che trova);

* Verifichiamo il titolo della pagina e il tag ``<h2>`` per assicurarci di essere sulla pagina giusta (avremmo anche potuto controllare che ci fosse una corrispondenza con la rotta);

* Infine, verifichiamo che ci sia un singolo commento sulla pagina. ``div:contains()`` non è un selettore CSS valido, ma Symfony ha alcune aggiunte interessanti, prese in prestito da jQuery.

Invece di cliccare sul testo (es: ``View``), avremmo potuto selezionare il link anche tramite un selettore CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Verifichiamo che il nuovo test sia verde:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Lavorare con un database di test
--------------------------------

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

Caricare le fixture per l'ambiente/database di ``test``:

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

Invio di un form in un test funzionale
--------------------------------------

Volete andare al livello successivo? Provate ad aggiungere un nuovo commento con una foto a una conferenza, da un test, simulando l'invio di un form. Sembra ambizioso, vero? Guardate il codice necessario: non più complesso di quello che abbiamo già scritto:

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

Per inviare un form tramite ``submitForm()``, trovare i nomi di degli input grazie al DevTools del browser, o tramite il pannello dei form del Profiler di Symfony. Si noti il riutilizzo intelligente dell'immagine in costruzione!

Eseguire nuovamente i test per verificare che sia tutto verde:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Uno dei vantaggi dell'utilizzo del database "dev" per i test è che è possibile controllare il risultato in un browser:

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Ricaricare le fixture
---------------------

.. index::
    single: Command;doctrine:fixtures:load

Eseguendo i test una seconda volta, dovrebbero fallire. Poiché adesso ci sono più commenti nel database, l'asserzione che controlla il numero di commenti fallisce. Bisogna resettare lo stato del database tra un'esecuzione e l'altra, ricaricando le fixture prima di ogni esecuzione:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatizzare il flusso di lavoro con un Makefile
-------------------------------------------------

.. index::
    single: Makefile

Ricordare una sequenza di comandi per eseguire i test è fastidioso. Andrebbe almeno riportata nella documentazione. Ma la documentazione dovrebbe essere l'ultima risorsa. Invece, che dire dell'automazione delle attività quotidiane? Potrebbe servire come documentazione, aiuterebbe a capirne il funzionamento ad altri sviluppatori e renderebbe la vita dello sviluppatore più semplice e veloce.

.. index::
    single: Command;doctrine:fixtures:load

Usare un ``Makefile`` è un modo per automatizzare i comandi:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

Si noti il parametro ``-n`` passato al comando Doctrine; è un'opzione globale dei comandi di Symfony che li rende non interattivi.

Ogni volta che si desidera eseguire i test, utilizzare ``make tests``:

.. code-block:: bash

    $ make tests

Ripristinare il database dopo ogni test
---------------------------------------

.. index::
    single: PHPUnit;Performance

Ripristinare il database dopo ogni test è una cosa positiva, ma avere test veramente indipendenti è ancora meglio. Non è desiderabile che un test si basi sui risultati dei precedenti. La modifica dell'ordine dei test non dovrebbe modificare il risultato. Come capiremo ora, per il momento non è così.

Spostare il test ``testConferencePage`` dopo ``testCommentSubmission``:

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

I test ora falliscono.

.. index::
    single: Doctrine;TestBundle

Per ripristinare il database tra un test e l'altro, installare DoctrineTestBundle:

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Sarà necessario confermare l'esecuzione della ricetta (in quanto non si tratta di un bundle "ufficialmente" supportato):

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

Abilitare il listener per PHPUnit:

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

Ci siamo. Tutte le modifiche fatte coi test sono ora automaticamente annullate alla fine di ogni test.

I test dovrebbero essere di nuovo verdi:

.. code-block:: bash

    $ make tests

Utilizzo di un browser reale per i test funzionali
--------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

I test funzionali usano un browser speciale che richiama direttamente Symfony. Ma si può anche usare un browser reale e vere chiamate HTTP usando Symfony Panther:

.. code-block:: bash

    $ symfony composer req panther --dev

È quindi possibile scrivere dei test che utilizzino un'istanza reale di Google Chrome con le seguenti modifiche:

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

La variabile d'ambiente ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contiene l'URL del server web locale.

Esecuzione di test funzionali "black box" con Blackfire
-------------------------------------------------------

Un altro modo per eseguire test funzionali è quello di utilizzare il `player di Blackfire  <https://blackfire.io/player>`_. Oltre a ciò che si può fare con i test funzionali, può anche eseguire test sulle prestazioni.

Fare riferimento al passo "Prestazioni" per approfondire.

.. sidebar:: Andare oltre

    * `Elenco delle assertion definite da Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ per i test funzionali;

    * `Documentazione di PHPUnit <https://phpunit.de/documentation.html>`_;

    * La libreria `Faker <https://github.com/fzaninotto/Faker>`_ per generare dati realistici nelle fixture;

    * La documentazione sul componente `CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * La libreria `Symfony Panther <https://github.com/symfony/panther>`_ per i test tramite browser e il web crawling nelle applicazioni Symfony;

    * La documentazione su `Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
