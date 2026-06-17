Pruebas
=======

.. index::
    single: PHPUnit

Ahora que empezamos a añadir más y más funcionalidades a la aplicación, es el momento ideal para hablar sobre las pruebas.

*Curiosidad* : Encontré un error al escribir las pruebas en este capítulo.

Symfony se basa en PHPUnit para las pruebas unitarias (*unit tests*). Vamos a instalarlo:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

Escribiendo pruebas unitarias
-----------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` es la primera clase para la que vamos a escribir las pruebas. Generemos una prueba unitaria:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

Probar el SpamChecker es un reto, ya que ciertamente no queremos llegar a la API de OpenAI: sería lento, costoso y las respuestas ni siquiera serían deterministas. Vamos a reemplazar la *platform* por una falsa.

.. index::
    single: Mock

Escribamos una primera prueba para cuando no se puede contactar con el modelo:

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

La clase ``InMemoryPlatform`` implementa la interfaz de la plataforma sin llamar a ninguna API externa. Dado un *callable*, puede simular cualquier comportamiento, incluyendo los fallos. La envolvemos en un ``Agent`` real para que la lógica del ``SpamChecker`` se pruebe de verdad.

Cuando el modelo no está disponible, los comentarios deben llegar a un moderador humano: la puntuación esperada es ``1``.

Ejecuta las pruebas para comprobar que se han superado:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

Añadamos pruebas para el *happy path*:

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

Los proveedores de datos de PHPUnit nos permiten reutilizar la misma lógica de prueba para varios casos de prueba.

Escribiendo pruebas funcionales para controladores
--------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

Probar controladores es un poco diferente a probar una clase "normal" de PHP, ya que queremos ejecutarlos en el contexto de una petición HTTP.

Creando una prueba funcional para el controlador Conference:

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

El uso de ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` en lugar de ``PHPUnit\Framework\TestCase`` como clase base para nuestros tests nos brinda una buena abstracción para las pruebas funcionales.

La variable ``$client`` simula un navegador. En lugar de hacer llamadas HTTP al servidor, llama directamente a la aplicación Symfony. Esta estrategia tiene varios beneficios: es mucho más rápida que tener viajes de ida y vuelta entre el cliente y el servidor, pero también permite que las pruebas puedan inspeccionar el estado de los servicios después de cada petición HTTP.

Esta primera prueba comprueba que la página de inicio devuelve una respuesta HTTP 200.

Con el fin de facilitarnos la vida, a PHPUnit se le han incorporado comprobaciones (*asserts*) del tipo ``assertResponseIsSuccessful`` (comprobar si la respuesta es exitosa). Existen muchas de estas comprobaciones definidas por Symfony.

.. tip::

    Hemos utilizado la URL ``/`` en lugar de generarla a través del enrutador. Esto se hace a propósito ya que probar las URLs de los usuarios finales es parte de lo que queremos probar. Si cambias la ruta, las pruebas fallarán para recordarte que, probablemente, deberías redirigir la URL antigua a la nueva para no entorpecer a los motores de búsqueda y los sitios web que enlazan con tu sitio web.

Configurando el Entorno de Pruebas
----------------------------------

.. index::
    single: Symfony Environments

Por defecto, las pruebas PHPUnit corren en el entorno ``test`` de Symfony definido en el fichero de configuración de PHPUnit:

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

Para hacer funcionar las pruebas, debemos definir el secreto ``OPENAI_API_KEY`` para este entorno ``test``:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

Trabajando con una base de datos de prueba
------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Como ya hemos visto, Symfony CLI expone automáticamente la variable de entorno ``DATABASE_URL``. Cuando ``APP_ENV`` es ``test``, como se establece al ejecutar PHPUnit, el nombre de la base de datos cambia de ``app`` a ``app_test`` para que las pruebas tengan su propia base de datos:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

Esto es muy importante ya que necesitaremos algunos datos estables para ejecutar nuestras pruebas y no queremos sobreescribir lo almacenado en la base de datos de desarrollo.

Antes de poder ejecutar la prueba, necesitamos "inicializar" la base de datos de ``test`` (crear la base de datos y migrarla):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    En Linux y sistemas operativos similares, puedes usar ``APP_ENV=test`` en
    lugar de ``--env=test``:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

Si ahora ejecutas las pruebas, PHPUnit ya no interactuará con tu base de datos de desarrollo. Para ejecutar solo las nuevas pruebas, indica la ruta de sus clases:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Cuando una prueba falla, puede ser útil una introspección del objeto ``Response``. Accede a él a través de ``$client->getResponse()`` y ``echo`` para ver su aspecto.

Definiendo factorías
--------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

Para poder probar la lista de comentarios, la paginación y el envío del formulario, necesitamos poblar la base de datos con algunos datos. Y para mantener las pruebas independientes entre sí, cada prueba debe crear el conjunto exacto de datos que necesita. Las *factorías de objetos* son la herramienta perfecta para esta tarea.

Instala Zenstruck Foundry:

.. code-block:: terminal

    $ symfony composer req foundry --dev

Genera una factoría para cada entidad que necesiten las pruebas:

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

Una factoría describe cómo construir una entidad válida: se genera un valor predeterminado para cada propiedad gracias a la librería Faker. Crear un objeto a través de una factoría también lo persiste. Ajusta los valores predeterminados de la conferencia para que sean más realistas:

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

Establecer el slug a ``-`` deja que el oyente de entidad que escribimos al añadir los slugs calcule el valor real: una conferencia creada con la ciudad ``Amsterdam`` y el año ``2019`` obtiene automáticamente el slug ``amsterdam-2019``.

Haz lo mismo para los comentarios:

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

Fíjate en el valor predeterminado de ``conference``: cuando se crea un comentario sin una conferencia explícita, Foundry crea una sobre la marcha.

Rastreo de un sitio web en pruebas funcionales
----------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Como hemos visto, el cliente HTTP utilizado en las pruebas simula un navegador, por lo que podemos navegar por la web como si estuviéramos utilizando un navegador sin interfaz gráfica.

Agrega una nueva prueba que haga clic en una página de la conferencia desde la página principal:

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

El trait ``Factories`` habilita las factorías en las pruebas, y ``ResetDatabase`` reinicia la base de datos al comienzo de cada ejecución de las pruebas.

Describamos lo que sucede en esta prueba en un lenguaje sencillo:

* La prueba crea el conjunto exacto de datos que necesita: dos conferencias y un comentario, a través de las factorías;

* Como en la primera prueba, vamos a la página de inicio;

* El método ``request()`` devuelve una instancia ``Crawler`` que ayuda a encontrar elementos en la página (como enlaces, formularios, o cualquier cosa a la que se pueda llegar con selectores CSS o XPath);

* Gracias a un selector CSS, nos aseguramos de que tenemos dos conferencias listadas en la página de inicio;

* Luego hacemos clic en el enlace "Ver" (como no puede hacer clic en más de un enlace a la vez, Symfony elige automáticamente el primero que encuentra);

* Verificamos el título de la página, la respuesta y el ``<h2>`` de la página para asegurarnos de que estamos en la página correcta (también podríamos haber comprobado que la ruta coincide);

* Finalmente, verificamos que hay 1 comentario en la página. ``div:contains()`` no es un selector de CSS válido, pero Symfony incluye algunas mejoras prestadas de jQuery.

En lugar de hacer clic en el texto (es decir, ``View``), también podríamos haber seleccionado el enlace a través de un selector CSS:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Comprueba que la nueva prueba está en verde:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Enviando un formulario en una prueba funcional
----------------------------------------------

¿Quieres pasar al siguiente nivel? Inténtalo añadiendo un nuevo comentario con una foto en una conferencia desde una prueba simulando el envío de un formulario. Eso parece ambicioso, ¿no? Mira el código necesario: no es más complejo que el que ya hemos escrito:

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

Para enviar un formulario a través de ``submitForm()``, busca los nombres de los *inputs* gracias al navegador DevTools o a través del panel del Symfony Profiler Form. ¡Observa la elegante reutilización de la imagen en construcción!

Vuelve a realizar las pruebas para comprobar que todo está en verde:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Si quieres comprobar el resultado en un navegador, para el servidor web y vuelve a ejecutarlo en el entorno ``test``:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Ejecutando las pruebas de nuevo
-------------------------------

Si ejecutas las pruebas por segunda vez, siguen pasando: el trait ``ResetDatabase`` reinicia la base de datos al comienzo de cada ejecución de las pruebas, y cada prueba crea el conjunto exacto de datos que necesita. No hay estado compartido ni restos de una ejecución anterior:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatizando el flujo de trabajo con un Makefile
-------------------------------------------------

.. index::
    single: Makefile

Tener que recordar la secuencia de comandos que ejecuta las pruebas es molesto. Debería, al menos, estar documentado. Pero la documentación debe ser el último recurso. En cambio, ¿qué hay de la automatización de las actividades cotidianas? Eso serviría como documentación, ayudaría a otros desarrolladores a descubrirlo y les haría la vida más fácil y rápida.

Usar un ``Makefile`` es una forma de automatizar comandos:

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

    En una regla Makefile, la sangría **debe** consistir en un único carácter de tabulación en lugar de espacios.

Observa el parámetro ``-n`` en el comando Doctrine; es un parámetro global de los comandos Symfony que los hace no interactivos.

Siempre que desees ejecutar las pruebas, utiliza ``make tests``:

.. code-block:: terminal

    $ make tests

Restableciendo la base de datos después de cada prueba
-------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Reiniciar la base de datos después de cada ejecución de pruebas es bueno, pero tener pruebas verdaderamente independientes es aún mejor. No queremos que una prueba se base en los resultados de las anteriores. Cambiar el orden de las pruebas no debe cambiar el resultado. Como vamos a descubrir ahora, éste no es el caso por el momento.

Mueve la prueba ``testConferencePage`` después de ``testCommentSubmission``:

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

Las pruebas ahora fallan.

.. index::
    single: Doctrine;TestBundle

Para restablecer la base de datos entre pruebas, instala DoctrineTestBundle:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

Deberás confirmar la ejecución de la receta (ya que no es un paquete soportado "oficialmente"):

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

Y hecho. Cualquier cambio realizado en las pruebas se retrotrae automáticamente al final de cada prueba.

Las pruebas deberían de nuevo estar en verde:

.. code-block:: terminal

    $ make tests

Usando un navegador real para pruebas funcionales
-------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Las pruebas funcionales utilizan un navegador especial que llama directamente a la capa de Symfony. Pero también puedes usar un navegador real y la capa HTTP real gracias a Symfony Panther:

.. code-block:: terminal

    $ symfony composer req panther --dev

Puedes escribir pruebas que utilicen un navegador Google Chrome real con los siguientes cambios:

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

La variable de entorno ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contiene la URL del servidor web local.

Eligiendo el tipo de prueba adecuado
------------------------------------

.. index::
    single: Command;make:test

Hasta ahora hemos creado tres tipos diferentes de pruebas. Aunque solo hemos usado el *maker bundle* para generar la clase de prueba unitaria, podríamos haberlo usado para generar también las otras clases de prueba:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

El *maker bundle* soporta la generación de los siguientes tipos de pruebas según cómo quieras probar tu aplicación:

* ``TestCase``: Pruebas básicas de PHPUnit;

* ``KernelTestCase``: Pruebas básicas que tienen acceso a los servicios de Symfony;

* ``WebTestCase``: Para ejecutar escenarios de tipo navegador, pero que no ejecutan código JavaScript;

* ``ApiTestCase``: Para ejecutar escenarios orientados a la API;

* ``PantherTestCase``: Para ejecutar escenarios e2e, usando un navegador real o un cliente HTTP y un servidor web real.

Ejecutando pruebas funcionales de caja negra (*Black Box*) con Blackfire
------------------------------------------------------------------------

Otra forma de realizar pruebas funcionales es utilizar el `reproductor Blackfire`_. Además de lo que puedes hacer con las pruebas funcionales, también puedes realizar pruebas de rendimiento.

Consulta el paso :doc:`Performance <29-performance>` para obtener más información.

.. sidebar:: Yendo más allá

    * `Lista de comprobaciones (*assertions*) definidas por Symfony`_ para pruebas funcionales;

    * `Documentación de PHPUnit`_ ;

    * La `documentación de Foundry`_ ;

    * La `biblioteca Faker`_ para generar datos realistas de *fixtures*;

    * La documentación del `componente CssSelector`_ ;

    * La biblioteca `Symfony Panther`_ para pruebas de navegadores y rastreo web en aplicaciones Symfony;

    * La `documentación Make/Makefile`_ .

.. _`reproductor Blackfire`: https://blackfire.io/player
.. _`Lista de comprobaciones (*assertions*) definidas por Symfony`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`Documentación de PHPUnit`: https://phpunit.de/documentation.html
.. _`documentación de Foundry`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`biblioteca Faker`: https://github.com/FakerPHP/Faker
.. _`componente CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`documentación Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
