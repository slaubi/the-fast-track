Testen
======

.. index::
    single: PHPUnit

Da wir nun mehr und mehr Funktionalität in die Anwendung einbauen, ist jetzt wahrscheinlich der richtige Zeitpunkt um über das Testen zu sprechen.

*Fun Fact*: Ich habe beim Schreiben der Tests in diesem Kapitel einen Fehler gefunden.

Symfony setzt bei Unit-Tests auf PHPUnit. Lass es uns installieren:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Unit-Tests schreiben
--------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` ist die erste Klasse, für die wir Tests schreiben werden. Generiere einen Unit-Test:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Das Testen des SpamCheckers ist eine Herausforderung, da wir die OpenAI-API sicherlich nicht aufrufen wollen: Das wäre langsam, teuer, und die Antworten wären nicht einmal deterministisch. Wir werden die *Plattform* durch eine falsche ersetzen.

.. index::
    single: Mock

Lasse uns einen ersten Test für den Fall schreiben, dass die API einen Fehler zurückgibt:

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

Die ``InMemoryPlatform``-Klasse implementiert das Plattform-Interface, ohne irgendeine externe API aufzurufen. Mit einem Callable kann sie jedes Verhalten simulieren, auch Fehler. Wir umschließen sie mit einem echten ``Agent``, damit die Logik des ``SpamChecker`` wirklich getestet wird.

Wenn das Modell ausgefallen ist, müssen Kommentare zu einem menschlichen Moderator gelangen: Der erwartete Score ist ``1``.

Führe die Tests aus, um sicherzustellen, dass sie erfolgreich sind:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Lasst uns Tests für den happy path hinzufügen:

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

Der PHPUnit Data Provider ermöglicht es uns, die gleiche Testlogik für mehrere Testfälle wiederzuverwenden.

Funktionale Tests für Controller schreiben
-------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Das Testen von Controllern ist etwas anders als das Testen einer "normalen" PHP-Klasse, da wir sie im Rahmen einer HTTP-Anfrage ausführen wollen.

Erstelle einen funktionalen Test für den Conference-Controller:

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

Wenn wir ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` anstelle von ``PHPUnit\Framework\TestCase`` als Basis-Klasse für unsere Tests nutzen, haben wir eine gute Abstraktion für unsere Funktionalen Tests.

Die ``$client``-Variable simuliert einen Browser. Anstatt jedoch HTTP-Anfragen an den Server zu senden, ruft dieser die Symfony-Anwendung direkt auf. Dieses Vorgehen hat mehrere Vorteile: Es ist viel schneller als eine tatsächliche Kommunikation zwischen Client und Server, und sie ermöglicht es auch, in den Tests den Zustand der Services nach jedem HTTP-Request zu überprüfen.

Dieser erste Test prüft, ob die Homepage eine HTTP-Response mit Status 200 zurückgibt.

Assertions wie ``assertResponseIsSuccessful`` werden zusätzlich zu PHPUnit hinzugefügt, um Dir die Arbeit zu erleichtern. Symfony stellt viele solcher Assertions zur Verfügung.

.. tip::

    Wir haben ``/`` fix als URL verwendet, anstatt sie über den Router zu generieren. Dies geschieht absichtlich, da das Testen von Produktiv-URLs Teil dessen ist, was wir testen wollen. Sobald Du den Routenpfad änderst, werden die Tests fehlschlagen und dich dadurch freundlich daran erinnern, dass Du die alte URL wahrscheinlich auf die neue umleiten solltest, um gegenüber Suchmaschinen und Websites, die auf Deine Website verweisen, nett zu sein.

Konfiguration der Test-Umgebung
-------------------------------

.. index::
    single: Symfony Environments

Standardmäßig werden PHPUnit-Tests in der ``test`` Symfony-Umgebung ausgeführt. Das ist in der PHPUnit-Konfigurations-Datei festgelegt:

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

Damit Tests funktionieren, müssen wir das ``OPENAI_API_KEY``-Secret für diese Umgebung festlegen:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Mit einer Testdatenbank arbeiten
--------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Wie wir schon eher gesehen haben, stellt die Symfony CLI automatisch die ``DATABASE_URL``-Environment-Variable (Umgebungsvariable) bereit. Genauso als wenn PHPUnit ausgeführt wurde und verändert damit den Datenbank-Namen von ``app`` zu ``app_test``, so dass die Tests ihre eigene Datenbank haben:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Das ist sehr wichtig, weil wir auch stabile Daten brauchen um unsere Tests auszuführen, und wir sicherlich nicht die Daten in unserer Development-Datenbank überschreiben wollen.

Bevor wir unsere Tests ausführen können, müssen wir die ``test``-Datenbank "initialisieren" (Datenbank erstellen und migrieren):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    On Linux and similiar OSes, you can use ``APP_ENV=test`` instead of
    ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Wenn Du nun die Tests ausführst, wird PHPUnit nicht mehr mit Deiner Development-Datenbank kommunizieren. Um nur die neuen Tests auszuführen, füge den Pfad zu deren Klassenpfad hinzu:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Wenn ein Test fehlschlägt, kann es sinnvoll sein, sich das Response-Objekt anzusehen. Greife über ``$client->getResponse()`` und ``echo`` darauf zu, um zu sehen, wie es aussieht.

Factories definieren
--------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

Um die Kommentarliste, Pagination und die Formularübermittlung testen zu können, müssen wir die Datenbank mit Daten befüllen. Und um die Tests voneinander unabhängig zu halten, sollte jeder Test genau den Datensatz erstellen, den er braucht. *Object Factories* sind das perfekte Werkzeug dafür.

Installiere Zenstruck Foundry:

.. code-block:: terminal

    $ symfony composer req foundry --dev

Generiere eine Factory für jede Entity, die die Tests brauchen:

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

Eine Factory beschreibt, wie eine gültige Entity gebaut wird: Für jedes Property wird dank der Faker-Bibliothek ein Standardwert generiert. Ein über eine Factory erstelltes Objekt wird auch gespeichert. Passe die Standardwerte der Konferenzen an, damit sie realistischer sind:

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

Den Slug auf ``-`` zu setzen lässt den Entity-Listener, den wir beim Hinzufügen der Slugs geschrieben haben, den echten Wert berechnen: Eine Konferenz mit der Stadt ``Amsterdam`` und dem Jahr ``2019`` erhält automatisch den Slug ``amsterdam-2019``.

Mache dasselbe für die Kommentare:

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

Beachte den Standardwert von ``conference``: Wird ein Kommentar ohne explizite Konferenz erstellt, erstellt Foundry auch eine Konferenz.

Eine Website in Funktionalen Tests crawlen
------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Wie wir gesehen haben, simuliert der in den Tests verwendete HTTP-Client einen Browser, sodass wir durch die Website navigieren können, als würden wir einen Headless-Browser verwenden.

Füge einen neuen Test hinzu, der von der Homepage aus auf eine Konferenzseite klickt:

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

Das ``Factories``-Trait aktiviert die Factories in den Tests, und ``ResetDatabase`` setzt die Datenbank zu Beginn jedes Testlaufs zurück.

Lasse uns in einfachen Worten beschreiben, was in diesem Test passiert:

* Der Test erstellt genau den Datensatz, den er braucht: zwei Konferenzen und einen Kommentar, über die Factories;

* Wie beim ersten Test gehen wir auf die Homepage;

* Die ``request()``-Methode gibt eine ``Crawler``-Instanz zurück, die hilft, Elemente auf der Seite zu finden (wie Links, Formulare oder alles, was Du mit CSS-Selektoren oder XPath erreichen kannst);

* Mit Hilfe eines CSS-Selektors prüfen wir, dass zwei Konferenzen auf der Homepage aufgelistet sind;

* Dann klicken wir auf den Link "View" (Symfony kann nicht mehr als einen Link gleichzeitig anklicken, darum wählt es automatisch den ersten, den es findet);

* Wir testen den Seitentitel, die Response und die Seitenüberschrift ``<h2>``, um sicher zu gehen, dass wir auf der richtigen Seite sind (wir hätten auch die zugehörige Route überprüfen können);

* Schließlich prüfen wir, dass es einen Kommentar auf der Seite gibt. ``div:contains()`` ist zwar kein gültiger CSS-Selektor, Symfony hat sich jedoch einige nützliche Ergänzungen von jQuery abgeschaut.

Anstatt auf den Text zu klicken (z.B. ``View``), hätten wir den Link auch über einen CSS-Selektor auswählen können:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Überprüfe, ob der neue Test grün ist:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Ein Formular in einem Funktionalen Test abschicken
--------------------------------------------------

Möchtest Du das nächste Level erreichen? Versuche, einen neuen Kommentar mit einem Foto auf einer Konferenz aus einem Test heraus hinzuzufügen, indem Du das Abschicken eines Formulares simulierst. Das scheint ehrgeizig zu sein, nicht wahr? Schaue Dir den benötigten Code an: nicht komplexer als das, was wir bereits geschrieben haben:

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

Um ein Formular über ``submitForm()`` abzuschicken, kannst Du die Namen der Felder über die Browser-DevTools oder über den Formular-Tab des Symfony Profilers finden. Beachte die clevere Wiederverwendung des "under construction"-Bildes!

Führe die Tests erneut durch, um sicherzustellen, dass alles grün ist:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Wenn Du das Ergebnis in einem Browser überprüfen willst, stoppe den Webserver und starte ihn noch einmal in der ``test``-Umgebung:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Die Tests erneut ausführen
--------------------------

Wenn Du die Tests ein zweites Mal ausführst, sind sie immer noch grün: Das ``ResetDatabase``-Trait setzt die Datenbank zu Beginn jedes Testlaufs zurück, und jeder Test erstellt genau den Datensatz, den er braucht. Es gibt keinen geteilten Zustand und keine Überbleibsel eines früheren Laufs:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Deinen Workflow mit einem Makefile automatisieren
-------------------------------------------------

.. index::
    single: Makefile

Es ist ärgerlich, sich eine Reihe von Befehlen merken zu müssen, um die Tests auszuführen. Dies sollte zumindest dokumentiert werden. Eine Dokumentation sollte jedoch nur der letzte Ausweg sein. Wie sieht es stattdessen mit der Automatisierung der täglichen Aktivitäten aus? Das würde als Dokumentation dienen, anderen Entwickler*innen helfen, sie zu entdecken und ihre Arbeit erleichtern und beschleunigen.

Die Verwendung von einem ``Makefile`` ist eine Möglichkeit, Befehle zu automatisieren:

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

    Nach einer Regel für Make-Dateien (Makefiles) **muss** die Einrückung aus einem einzelnen Tabulator-Zeichen anstelle von Leerzeichen bestehen.

Beachte das ``-n``-Flag des Doctrine Befehls; es ist ein globales Flag für Symfony Befehle, das sie nicht interaktiv macht.

Wann immer Du die Tests ausführen möchtest, verwende ``make tests``:

.. code-block:: terminal

    $ make tests

Die Datenbank nach jedem Test zurücksetzen
-------------------------------------------

.. index::
    single: PHPUnit;Performance

Das Zurücksetzen der Datenbank nach jedem Testlauf ist schön, aber wirklich unabhängige Tests sind noch besser. Wir wollen nicht, dass sich ein Test auf die Ergebnisse der vorherigen stützt. Eine Änderung der Reihenfolge der Tests sollte das Ergebnis nicht verändern. Wie wir jetzt herausfinden werden, ist dies im Moment nicht der Fall.

Verschiebe den ``testConferencePage``-Test hinter den ``testCommentSubmission``-Test:

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

Jetzt schlagen die Tests fehl.

.. index::
    single: Doctrine;TestBundle

Installiere das DoctrineTestBundle, um die Datenbank zwischen den Tests zurückzusetzen:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

Du musst die Ausführung des Recipes bestätigen (da es sich nicht um ein "offiziell" unterstütztes Bundle handelt):

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

Und fertig. Alle Änderungen, die in Tests vorgenommen werden, werden nun am Ende jedes Tests automatisch zurückgesetzt.

Die Tests sollten wieder grün sein:

.. code-block:: terminal

    $ make tests

Einen echten Browser für Funktionale Tests verwenden
-----------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Funktionale Tests verwenden einen speziellen Browser, der den Symfony-Layer direkt aufruft. Aber Du kannst auch einen echten Browser und den echten HTTP-Layer dank Symfony Panther verwenden:

.. code-block:: terminal

    $ symfony composer req panther --dev

Du kannst dann Tests schreiben, die einen echten Google Chrome-Browser verwenden. Dazu benötigst Du die folgenden Änderungen:

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

Die Environment-Variable ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` enthält die URL des lokalen Webservers.

Den richtigen Test Typen wählen
--------------------------------

.. index::
    single: Command;make:test

Bisher haben wir drei verschiedene Test Typen erstellt. Während wir das Maker-Bundle nur genutzt haben um Unit-Test-Klassen zu generieren, könnten wir es auch zur Generierung der anderen Test-Klassen nutzen:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

Das Maker-Bundle unterstützt die Generierung der folgenden Test Typen, abhängig davon wie Du Deine Applikation testen möchtest:

* ``TestCase``: Standard PHPUnit-Tests;

* ``KernelTestCase``: Standard Tests die Zugang zu Symfony Diensten haben;

* ``WebTestCase``: um Browser-ähnliche Szenarios, aber ohne Javascript Code auszuführen;

* ``ApiTestCase``: für API-orientierte Szenarios;

* ``PantherTestCase``: für End-zu-End Szenarios; welche einen echten Browser oder HTTP-Client und einen echten Web-Server nutzen.

Funktionale "Black Box"-Tests mit Blackfire durchführen
--------------------------------------------------------

Eine weitere Möglichkeit, Funktionale Tests durchzuführen, ist die Verwendung des `Blackfire-Players`_. Zusätzlich zu dem, was Du mit Funktionalen Tests machen kannst, kann der Blackfire-Player auch Performance Tests durchführen.

Schau Dir den Schritt über :doc:`Performance <29-performance>` an, um mehr zu erfahren.

.. sidebar:: Weiterführendes

    * `Liste der von Symfony definierten Assertions`_ für Funktionale Tests;

    * `PHPUnit Dokumentation`_;

    * Die `Foundry-Dokumentation`_;

    * Die `Faker Bibliothek`_ zur Erstellung realistischer Fixtures;

    * Die `CssSelector Component Dokumentation`_;

    * Die `Symfony Panther`_ Bibliothek für Browsertests und Webcrawling in Symfony-Anwendungen;

    * Die `Make/Makefile Dokumentation`_.

.. _`Blackfire-Players`: https://blackfire.io/player
.. _`Liste der von Symfony definierten Assertions`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`PHPUnit Dokumentation`: https://phpunit.de/documentation.html
.. _`Foundry-Dokumentation`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`Faker Bibliothek`: https://github.com/FakerPHP/Faker
.. _`CssSelector Component Dokumentation`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`Make/Makefile Dokumentation`: https://www.gnu.org/software/make/manual/make.html
