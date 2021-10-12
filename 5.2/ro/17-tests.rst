Testare
=======

.. index::
    single: PHPUnit

Din moment ce adăugăm din ce în ce mai multe funcționalități în aplicație, este probabil momentul potrivit pentru a vorbi despre testare.

*Fapt amuzant*: am găsit o eroare în timp ce scriam testele în acest capitol.

Symfony se bazează pe PHPUnit pentru teste unitare. Să-l instalăm:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Elaborarea testelor unitare
---------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` este prima clasă pentru care vom scrie teste. Generează un test unitar:

.. code-block:: bash

    $ symfony console make:test TestCase SpamCheckerTest

Testarea SpamChecker este o provocare, deoarece cu siguranță nu vrem să atingem API-ul Akismet. Drept alternativă vom *simula* API-ul.

.. index::
    single: Mock

Să elaborăm un prim test pentru situația în care API-ul returnează o eroare:

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

Clasa ``MockHttpClient`` face posibilă simularea oricărui server HTTP. Este nevoie de o serie de instanțe ``MockResponse`` care conțin anteturile preconizate ale corpului și răspunsului.

Apoi, apelăm la metoda ``getSpamScore()`` și verificăm dacă o excepție este aruncată prin metoda ``expectException()`` a PHPUnit.

Execută testele pentru a verifica dacă acestea trec:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Să adăugăm teste pentru calea fericită:

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

Furnizorii de date PHPUnit ne permit să reutilizăm aceeași logică de testare pentru mai multe cazuri de testare.

Redactarea testelor funcționale pentru controlere
--------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Testarea controlerelor este puțin diferită de testarea unei clase PHP „obișnuite”, deoarece dorim să le executăm în contextul unei solicitări HTTP.

Creează un test funcțional pentru controlerul Conference:

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

Folosirea ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` în loc de ``PHPUnit\Framework\TestCase`` precum clasă de bază pentru testele noastre ne oferă o abstractizare bună pentru testele funcționale.

Variabila ``$client`` simulează un browser. Totuși, în loc să efectueze apeluri HTTP către server, apelează direct aplicația Symfony. Această strategie are mai multe avantaje: este mult mai rapidă decât efectuarea transmiterea mesajelor dus-întors între client și server, dar permite și testele să introspecte starea serviciilor după fiecare solicitare HTTP.

Acest prim test verifică dacă pagina principală returnează un răspuns HTTP 200.

Validări precum ``assertResponseIsSuccessful`` sunt adăugate peste cele existente în PHPUnit pentru a-ți ușura munca. Există multe astfel de validări definite de Symfony.

.. tip::

    Am folosit ``/`` pentru URL în loc să-l generăm prin router. Acest lucru se face în mod intenționat, deoarece testarea adreselor URL ale utilizatorului final face parte din ceea ce dorim să testăm. Dacă schimbi calea rutelor, testele vor da eroare, amintindu-ți că ar trebui să redirecționezi vechiul URL-ul vechi către unul nou, pentru a păstra funcționale legăturile din motoarele de căutare și de la alte site-urile web.

.. note::

    Am fi putut genera testul prin pachetul maker:

    .. code-block:: bash

        $ symfony console make:test WebTestCase Controller\\ConferenceController

Configurarea mediului de testare
--------------------------------

.. index::
    single: Symfony Environments

În mod implicit, testele PHPUnit sunt executate în mediul Symfony ``test`` după cum este definit în fișierul de configurare PHPUnit.

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

Pentru ca testele să funcționeze, trebuie să setăm secretul ``AKISMET_KEY`` pentru acest mediu ``test``:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

.. note::

    După cum am văzut într-un capitol anterior, ``APP_ENV=test`` înseamnă că variabila de mediu ``APP_ENV`` este setată pentru contextul comenzii. Pe Windows folosește ``--env=test``: ``symfony console secrets:set AKISMET_KEY --env=test``

Lucrul cu o bază de date de testare
------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

După cum am văzut deja, Symfony CLI expune automat variabila de mediu ``DATABASE_URL``. Când ``APP_ENV`` este ``test``, așa cum este setat când se execută PHPUnit, acesta schimbă numele bazei de date de la ``main`` la ``main_test``, astfel încât testele să aibă propria lor bază de date. Acest lucru este foarte important, deoarece vom avea nevoie de niște date stabile pentru a rula testele noastre și cu siguranță nu vrem să anulăm ceea ce am stocat în baza de date de dezvoltare.

Înainte de a putea rula testul, trebuie să „inițializăm” baza de date ``test`` (creează baza de date și migrează-o):

.. code-block:: bash

    $ APP_ENV=test symfony console doctrine:database:create
    $ APP_ENV=test symfony console doctrine:migrations:migrate -n

Dacă rulezi acum testele, PHPUnit nu va mai interacționa cu baza de date de dezvoltare. Pentru a rula doar teste noi, adaugă calea către clasele lor:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Observă că setăm `APP_ENV`` în mod explicit chiar și atunci când executăm PHPUnit pentru a permite Symfony CLI să seteze numele bazei de date la ``main_test``.

.. tip::

    Când un test nu trece, poate fi utilă analizarea obiectului Response. Accesează-l prin ``$client->getResponse()`` și ``echo`` pentru a vedea cum arată.

Definirea datelor de test
-------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Pentru a putea testa lista de comentarii, paginarea și trimiterea formularului, trebuie să populăm baza de date cu ceva informații. Și dorim ca datele să fie aceleași între teste pentru ca acestea să fie executate cu succes. Datele de testare sunt exact ceea ce avem nevoie.

Instalează pachetul Doctrine Fixtures:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

În timpul instalării a fost creat un nou director ``src/DataFixtures/``, cu o clasă exemplu, gata de a fi personalizată. Adaugă acum două conferințe și un singur comentariu:

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

Când vom încărca datele de testare, toate datele vor fi eliminate; inclusiv utilizatorul admin. Pentru a evita acest lucru, să adăugăm utilizatorul administrator în test:

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

    Dacă nu îți amintești ce serviciu trebuie să utilizezi pentru o sarcină dată, utilizează comanda ``debug:autowiring`` urmată de un cuvânt cheie:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Încărcarea datelor de testare
-------------------------------

.. index:: ! Command;doctrine:fixtures:load

Încarcă datele de testare pentru mediul/baza de date ``test``:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load

Navigarea unui site web în teste funcționale
----------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Așa cum am văzut, clientul HTTP utilizat în teste simulează un browser, astfel încât putem naviga prin intermediul site-ului ca și cum am folosi un browser headless.

Adaugă un test nou care face clic pe o pagină de conferință din pagina principală:

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

Să descriem ce se întâmplă în acest test într-un limbaj simplu:

* Ca și primul test, navigăm spre pagina principală;

* Metoda ``request()`` returnează o instanță` ``Crawler`` care ajută la găsirea de elemente pe pagină (cum ar fi link-uri, formulare sau orice ce poate fi accesat cu selectoare CSS sau XPath);

* Mulțumită unui selector CSS, afirmăm că avem două conferințe listate pe pagina principală;

* Facem clic apoi pe linkul „View” (deoarece nu se poate face clic pe mai multe linkuri simultan, Symfony alege automat primul pe care îl găsește);

* Verificăm titlul paginii, răspunsul și pagina ``<h2>`` pentru a fi siguri că suntem pe pagina corectă (am fi putut verifica și traseul corespunzător);

* În cele din urmă, verificăm că există 1 comentariu pe pagină. ``div: contains()`` nu este un selector CSS valid, dar Symfony are câteva completări frumoase, împrumutate de la jQuery.

În loc să facem clic pe text (adică ``View``), am fi putut selecta și linkul prin intermediul unui selector CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Verificați dacă noul test se execută cu succes:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Trimiterea unui formular într-un test funcțional
--------------------------------------------------

Vrei să treci la nivelul următor? Încearcă să adaugi un comentariu nou cu o fotografie la o conferință, de la un test, simulând o expediere a formularului. Asta pare ambițios, nu-i așa? Uită-te la codul necesar: nu e mai complex decât ceea ce am elaborat deja:

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

Pentru a expedia un formular prin ``submitForm()``, găsește numele de intrare datorită instrumentului DevTools a browser-ului sau prin panoul Symfony Profiler Form. Observă reutilizarea inteligentă a imaginii „În construcție”!

Execută testele din nou pentru a verifica dacă validările trec cu succes:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Dacă vrei să verifici rezultatul într-un browser, oprește serverul Web și pornește-l din nou pentru mediul „test”:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Reîncărcarea datelor de testare
---------------------------------

.. index::
    single: Command;doctrine:fixtures:load

Dacă execuți testele a doua oară, acestea ar trebui să eșueze. Deoarece acum există mai multe comentarii în baza de date, validarea care verifică numărul de comentarii este invalidă. Trebuie să resetăm starea bazei de date între fiecare execuție, reîncărcând datele de test înainte de fiecare execuție:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load
    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatizarea fluxului de lucru cu un Makefile
----------------------------------------------

.. index::
    single: Makefile

Necesitatea de a-ți aminti o secvență de comenzi pentru a rula testele este incomodă. Acest lucru ar trebui cel puțin să fie documentat. Dar documentația ar trebui să fie o ultimă soluție. În schimb, cum rămâne cu automatizarea activităților de zi cu zi? Aceasta ar servi drept documentare, ar ajuta alți dezvoltatori și ar face viața dezvoltatorilor mai ușoară și rapidă.

.. index::
    single: Command;doctrine:fixtures:load

Folosirea unui ``Makefile`` este o modalitate de a automatiza comenzile:

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

    Într-o regulă Makefile, indentarea **trebuie** să fie formată dintr-un singur caracter tab în loc de spații.

Rețineți opțiunea ``-n`` pentru comanda Doctrine; este o opțiune globală pentru comenzile Symfony care le face să nu fie interactive.

Ori de câte ori dorești să rulezi testele, folosește ``make tests``:

.. code-block:: bash

    $ make tests

Resetarea bazei de date după fiecare test
------------------------------------------

.. index::
    single: PHPUnit;Performance

Resetarea bazei de date după fiecare testare este plăcută, dar executarea testelor independent este și mai bine. Nu dorim ca un test să se bazeze pe rezultatele celor anterioare. Modificarea ordinii testelor nu ar trebui să schimbe rezultatul. După cum o să ne dăm seama acum, nu este cazul deocamdată.

Mută testul ``testConferencePage`` după testul ``testCommentSubmission``:

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

Testele acum eșuează.

.. index::
    single: Doctrine;TestBundle

Pentru a reseta baza de date între teste, instalează DoctrineTestBundle:

.. code-block:: bash
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: bash

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Va trebui să confirmi execuția rețetei (deoarece nu este un pachet „oficial” acceptat):

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

Activează ascultătorul PHPUnit:

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

Și gata. Orice modificări efectuate în teste sunt acum revizuite automat la sfârșitul fiecărui test.

Testele ar trebui să fie din nou validate:

.. code-block:: bash

    $ make tests

Utilizarea unui browser real pentru teste funcționale
------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Testele funcționale utilizează un browser special care apelează direct stratul Symfony. Dar, de asemenea, poți utiliza un browser real și stratul HTTP real datorită Symfony Panther:

.. code-block:: bash

    $ symfony composer req panther --dev

Poți scrie teste care utilizează un browser Google Chrome real cu următoarele modificări:

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

Variabila de mediu ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` conține adresa URL a serverului web local.

Executarea testelor funcționale în formatul cutiei negre cu Blackfire
-----------------------------------------------------------------------

Un alt mod de a rula teste funcționale este de a utiliza `Blackfire player <https://blackfire.io/player>`_. Pe lângă ceea ce poți face cu teste funcționale, poate efectua și teste de performanță.

Consultă punctul „Performanță” pentru a afla mai multe.

.. sidebar:: Mergând mai departe

    * `Lista validărilor definite de Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ pentru teste funcționale;

    * `Documentația PHPUnit <https://phpunit.de/documentation.html>`_;

    * `Biblioteca Faker <https://github.com/FakerPHP/Faker>`_ pentru a genera date de test realiste;

    * `Documentația componentei CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * Bibliotecă `Symfony Panther <https://github.com/symfony/panther>`_ bibliotecă pentru testarea browserului și accesare web în aplicațiile Symfony;

    * `Documentația Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
