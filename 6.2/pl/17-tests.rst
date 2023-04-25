Testowanie
==========

.. index::
    single: PHPUnit

Ponieważ zaczynamy dodawać coraz więcej i więcej funkcji do naszej aplikacji, jest to prawdopodobnie dobry czas, by poruszyć temat testowania.

*Ciekawostka*: Znalazłem błąd podczas pisania testów do tego rozdziału.

Symfony używa PHPUnit do testów jednostkowych. Zainstalujmy go:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Pisanie testów jednostkowych
-----------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` będzie pierwszą klasą, dla której napiszemy testy. Wygeneruj test jednostkowy:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Testowanie SpamCheckera jest niemałym wyzwaniem, ponieważ nie chcemy się łączyć z prawdziwym API Akismet. Będziemy musieli stworzyć *atrapę* (ang. mock).

.. index::
    single: Mock

Napiszmy nasz pierwszy test dla przypadku, kiedy API zwraca błąd w odpowiedzi:

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
    +    public function testSpamScoreWithInvalidRequest(): void
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

Klasa ``MockHttpClient`` pozwala na stworzenie atrapy (ang. "mock") dla dowolnego serwera HTTP. Jako argument przyjmuje ona tablicę instancji ``MockResponse`` z oczekiwaną odpowiedzią i nagłówkami.

Następnie wywołujemy ``getSpamScore()`` i przez metodę ``expectException()`` w PHPUnit sprawdzamy, czy otrzymaliśmy wyjątek.

Uruchom testy, by sprawdzić, czy wykonują się poprawnie:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;@dataProvider

Dodajmy testy dla przypadku, gdy wszystko przejdzie bezbłędnie (tzw. happy path):

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
    +     * @dataProvider provideComments
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
    +    public static function provideComments(): iterable
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

Dostawcy danych (ang. data providers) w PHPUnit pozwalają nam na użycie jednego schematu testu dla wielu przypadków.

Pisanie testów funkcjonalnych dla kontrolerów
-----------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Testowanie kontrolerów jest nieco inne niż testowanie "zwykłej" klasy PHP, ponieważ chcemy je uruchamiać w kontekście żądania HTTP.

Stwórz test funkcjonalny dla kontrolera Conference:

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

Użycie ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` zamiast ``PHPUnit\Framework\TestCase`` jako klasy bazowej dla naszych testów, pozwala uzyskać lepszą abstrakcję dla naszych testów funkcjonalnych.

Zmienna ``$client`` symuluje przeglądarkę. Jednak zamiast łączyć się z serwerem przez HTTP, wykonuje ona kod bezpośrednio wewnątrz aplikacji Symfony. Strategia ta ma kilka zalet: jest znacznie szybsza niż wymiana danych pomiędzy klientem a serwerem oraz pozwala testom na sprawdzenie stanu serwisów po każdym żądaniu HTTP.

Ten pierwszy test sprawdza, czy strona główna zwraca 200 jako kod statusu odpowiedzi HTTP.

Symfony rozszerza PHPUnit dodając asercje typu ``assertResponseIsSuccessful``, by ułatwić nam pracę. Istnieje wiele takich asercji zdefiniowanych przez Symfony.

.. tip::

    Użyliśmy ``/`` jako URL zamiast generowania go przez router. Jest to celowy zabieg, ponieważ testowanie adresów URL użytkownika końcowego jest częścią tego, co chcemy przetestować. Jeśli w przyszłości zmieni się ścieżka, testy przypomną Ci, że prawdopodobnie powinno zostać dodane przekierowanie ze starego adresu na nowy, aby być przyjaznym dla wyszukiwarek i stron internetowych, które odsyłają do Twojej strony.

Konfigurowanie środowiska testowego
------------------------------------

.. index::
    single: Symfony Environments

Domyślnie, PHPUnit w środowisku Symfony o nazwie ``test``, tak jak zostało to ustawione w pliku konfiguracyjnym PHPUnit:

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

Aby nasz test zadziałał, musimy ustawić poufny klucz ``AKISMET_KEY`` dla środowiska ``test``:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY --env=test

Praca z testową bazą danych
-----------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Widzieliśmy również, że narzędzie Symfony CLI automatycznie udostępnia zmienną środowiskową ``DATABASE_URL``. Kiedy zmienna ``APP_ENV`` jest ustawiona na ``test``, podobnie jak wtedy, kiedy uruchamialiśmy PHPUnit, zmienia nazwę bazy danych z ``app`` na ``app_test``, aby testy miały swoją własną bazę danych:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Jest to bardzo ważne, ponieważ będziemy potrzebować danych do przeprowadzenia naszych testów, ale też nie chcemy nadpisywać tego, co przechowujemy w bazie dla środowiska deweloperskiego.

Zanim będziemy mogli uruchomić test, musimy zainicjalizować bazę ``test`` (stworzyć tę bazę i wykonać migracje):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Jeżeli teraz uruchomisz testy, PHPUnit nie będzie używał Twojej deweloperskiej bazy danych. Aby uruchomić tylko nowe testy, przekaż ich ścieżki do odpowiednich klas (ang. class path):

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Kiedy test kończy się niepowodzeniem, przydatny może okazać się wgląd do obiektu Response. Możesz się do niego dostać przez ``$client->getResponse()`` i użyć ``echo`` by sprawdzić jak on wygląda.

Definiowanie danych testowych (ang. fixtures)
---------------------------------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Aby móc przetestować listę komentarzy, stronicowanie i wysyłanie formularza, musimy wypełnić bazę danych jakimiś danymi. Dodatkowo chcemy, aby te dane były niezmienne pomiędzy poszczególnymi testami. Dane testowe (ang. fixtures) są dokładnie tym, czego potrzebujemy.

Zainstaluj Doctrine Fixtures Bundle:

.. code-block:: terminal

    $ symfony composer req orm-fixtures --dev

Podczas instalacji został utworzony nowy folder ``src/DataFixtures/`` z przykładową klasą, gotową do zmodyfikowania. Dodajmy dwie konferencje i jeden komentarz:

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

W trakcie ładowania danych testowych wszystkie dotychczasowe dane są usuwane. Aby uniknąć usunięcia konta administracyjnego, musimy dodać je do naszych danych testowych (ang. fixtures):

.. code-block:: diff

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
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

    Jeśli nie pamiętasz, którego serwisu musisz użyć do danego zadania, użyj ``debug:autowiring`` ze słowami kluczowymi:

    .. code-block:: terminal

        $ symfony console debug:autowiring hasher

Ładowanie danych testowych (ang. fixtures)
-------------------------------------------

.. index:: ! Command;doctrine:fixtures:load

Załaduj dane testowe dla środowiska/bazy danych ``test``:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:fixtures:load --env=test

Przeszukiwanie (ang. crawling) strony w testach funkcjonalnych
--------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Jak widzieliśmy, klient HTTP użyty w testach symuluje przeglądarkę, dzięki czemu możemy poruszać się po stronie tak, jakbyśmy korzystali z przeglądarki bez interfejsu.

Dodaj nowy test, który kliknie w odnośnik do strony konferencji na stronie głównej:

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

Opiszmy prostymi słowami, co się dzieje w tym teście:

* Tak jak w pierwszym teście, otwieramy stronę główną;

* Metoda ``request()`` zwraca instancję klasy ``Crawler``, która pomaga znaleźć elementy na stronie (takie jak odnośniki, formularze lub cokolwiek, do czego można dotrzeć za pomocą selektora CSS lub XPath);

* Dzięki selektorowi CSS sprawdzamy, czy na stronie głównej są wyświetlone dwie konferencje;

* Następnie klikamy w odnośnik "View" (jako że nie można kliknąć w więcej niż jeden odnośnik na raz, Symfony automatycznie wybierze pierwszy, który znajdzie);

* Sprawdzamy tytuł strony, odpowiedź serwera i ``<h2>`` strony, by upewnić się, że znajdujemy się na właściwej (moglibyśmy również sprawdzić, czy ścieżka się zgadza);

* I w końcu sprawdzamy, czy na stronie znajduje się jeden komentarz. ``div:contains()`` nie jest poprawnym selektorem CSS, jednak Symfony posiada parę takich dodatków zapożyczonych z jQuery.

Zamiast klikać w tekst (np. ``View``), mogliśmy również wybrać link za pomocą selektora CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Sprawdź czy test przechodzi "na zielono":

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Wysyłanie formularza przez test funkcjonalny
---------------------------------------------

Chcesz wejść na wyższy poziom? Spróbuj dodać nowy komentarz ze zdjęciem przez formularz konferencji, symulując wysłanie formularza. Ambitne, prawda? Spójrz na potrzebny kod: nie jest bardziej skomplikowany niż ten, który już napisaliśmy:

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

Aby wysłać formularz za pośrednictwem ``submitForm()``, znajdź nazwy elementów formularza, używając narzędzi deweloperskich w przeglądarce lub zakładki Form w panelu Symfony Profiler. Zwróć uwagę na przemyślane ponowne wykorzystanie obrazka "w budowie"!

Uruchom testy jeszcze raz, aby upewnić się, że wszystkie przechodzą "na zielono":

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Jeżeli chcesz sprawdzić wyniki w przeglądarce, zatrzymaj serwer WWW i uruchom go ponownie, ale tym razem dla środowiska ``test``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Ponowne ładowanie danych testowych (ang. fixtures)
---------------------------------------------------

.. index::
    single: Command;doctrine:fixtures:load

Jeśli uruchomisz testy drugi raz, powinny one zakończyć się niepowodzeniem. Ponieważ w bazie danych znajduje się teraz więcej komentarzy, asercja sprawdzająca liczbę komentarzy nie będzie działać poprawnie. Musimy zresetować stan bazy danych pomiędzy każdym uruchomieniem poprzez załadowanie danych testowych (ang. fixtures):

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:fixtures:load --env=test
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatyzacja pracy (ang. workflow) z pomocą pliku Makefile
------------------------------------------------------------

.. index::
    single: Makefile

Zapamiętywanie sekwencji poleceń do przeprowadzenia testów jest irytujące. Jednym z rozwiązań może być spisanie ich, jednak dokumentacja powinna być ostatecznością. Może zamiast tego powinniśmy zautomatyzować tę codzienną czynność? Byłaby to forma dokumentacji oraz ułatwienie i przyspieszenie pracy dla innych.

.. index::
    single: Command;doctrine:fixtures:load

Używanie ``Makefile`` jest jednym ze sposobów zautomatyzowania poleceń:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:database:drop --force --env=test || true
    	symfony console doctrine:database:create --env=test
    	symfony console doctrine:migrations:migrate -n --env=test
    	symfony console doctrine:fixtures:load -n --env=test
    	symfony php bin/phpunit $@
    .PHONY: tests

.. warning::

    Wcięcia w regułach pliku Makefile **muszą** składać się z pojedynczego znaku tabulacji zamiast spacji.

Zwróć uwagę na flagę ``-n`` przy poleceniu Doctrine; jest to globalna flaga dla poleceń Symfony, która sprawia, że nie są one interaktywne.

Kiedykolwiek będziesz chciał uruchomić testy, użyj ``make tests``:

.. code-block:: terminal

    $ make tests

Resetowanie bazy danych po każdym teście
------------------------------------------

.. index::
    single: PHPUnit;Performance

Resetowanie bazy danych po każdym teście jest w porządku, ale używanie prawdziwie niezależnych testów jest jeszcze lepsze. Nie chcemy przecież, żeby jakikolwiek test opierał się na poprzednich wynikach. Zmiana kolejności testów nie powinna mieć wpływu na rezultat. Jak się zaraz przekonamy, nie jest to póki co prawdą.

Przenieś test ``testConferencePage`` tak, by znajdował się za testem ``testCommentSubmission``:

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

Teraz testy zwracają błąd.

.. index::
    single: Doctrine;TestBundle

Aby resetować bazę danych pomiędzy testami, zainstaluj Doctrine Test Bundle:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Będziesz musiał potwierdzić wykonanie przepisu (ang. recipe), ponieważ nie jest to "oficjalnie" obsługiwany pakiet (ang. bundle):

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

Dodaj nasłuchiwacz PHPUnit (ang. listener):

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

I gotowe. Wszelkie zmiany dokonywane przez testy będą teraz automatycznie cofane po zakończeniu każdego z nich.

Testy znowu powinny świecić się na zielono:

.. code-block:: terminal

    $ make tests

Korzystanie z prawdziwej przeglądarki do testów funkcjonalnych
----------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Testy funkcjonalne wykorzystują specjalną przeglądarkę, która bezpośrednio wywołuje warstwę Symfony. Jednak dzięki Symfony Panther, możesz również użyć prawdziwej przeglądarki i prawdziwej warstwy HTTP:

.. code-block:: terminal

    $ symfony composer req panther --dev

Następnie możesz pisać testy z użyciem prawdziwego Google Chrome z następującymi zmianami:

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

Zmienna środowiskowa ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` zawiera adres URL lokalnego serwera WWW.

Wybór odpowiedniego typu testu
-------------------------------

.. index::
    single: Command;make:test

Do tej pory stworzyliśmy trzy różne rodzaje testów. Chociaż użyliśmy pakietu maker tylko do wygenerowania klasy testów jednostkowych, moglibyśmy użyć go również do wygenerowania innych klas testowych:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

Maker bundle umożliwia generowanie następujących typów testów, w zależności od tego, jak chcesz przetestować swoją aplikację:

* ``TestCase``: Podstawowe testy PHPUnit;

* ``KernelTestCase``: Podstawowe testy, które mają dostęp do usług Symfony;

* `WebTestCase``: Aby uruchomić scenariusze oparte o zachowanie przeglądarki, ale które nie wykonują kodu JavaScript;

* ``ApiTestCase``: Aby uruchomić scenariusze testów oparte o API;

* `PantherTestCase``: Aby uruchomić scenariusze e2e, używając prawdziwej przeglądarki lub klienta HTTP i prawdziwego serwera WWW.

Uruchamianie czarnoskrzynkowych testów funkcjonalnych (ang. black box) przy użyciu Blackfire
----------------------------------------------------------------------------------------------

Innym sposobem na przeprowadzenie testów funkcjonalnych jest użycie `Blackfire player`_. Oprócz zwykłych testów funkcjonalnych, potrafi on również przeprowadzać testy wydajnościowe.

Aby dowiedzieć się więcej, zapoznaj się z rozdziałem :doc:`Wydajność <29-performance>`.

.. sidebar:: Idąc dalej

    * `Lista asercji definiowanych przez Symfony`_ dla testów funkcjonalnych;

    * `Dokumentacja PHPUnit`_

    * `Biblioteka Faker`_ do generowania danych testowych;

    * `Dokumentacja komponentu CssSelector`_;

    * `Symfony Panther`_  biblioteka do testowania przez przeglądarkę i przeszukiwania (ang. crawling) stron internetowych w aplikacjach opartych na Symfony;

    * `Dokumentacja Make/Makefile`_.

.. _`Blackfire player`: https://blackfire.io/player
.. _`Lista asercji definiowanych przez Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`Dokumentacja PHPUnit`: https://phpunit.de/documentation.html
.. _`Biblioteka Faker`: https://github.com/FakerPHP/Faker
.. _`Dokumentacja komponentu CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`Dokumentacja Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
