Pruebas
=======

.. index::
    single: PHPUnit

Ahora que empezamos a añadir más y más funcionalidades a la aplicación, es el momento ideal para hablar sobre las pruebas.

*Curiosidad* : Encontré un error al escribir las pruebas en este capítulo.

Symfony se basa en PHPUnit para las pruebas unitarias (*unit tests*). Vamos a instalarlo:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Escribiendo pruebas unitarias
-----------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` es la primera clase para la que vamos a escribir las pruebas. Generemos una prueba unitaria:

.. code-block:: bash

    $ symfony console make:test TestCase SpamCheckerTest

Probar el SpamChecker es un reto, ya que ciertamente no queremos llegar a la API de Akismet. Vamos a "*burlarnos*" de la API. En el ámbito de las pruebas, un *mock* (burla en inglés) es una clase o método propio que suplanta a uno real.

.. index::
    single: Mock

Escribamos una primera prueba para cuando la API devuelva un error:

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

La clase ``MockHttpClient`` permite hacer un *mock* de cualquier servidor HTTP. Para ello toma un *array* de instancias ``MockResponse`` que contienen el cuerpo esperado y las cabeceras de cada respuesta.

Más tarde llamamos al método ``getSpamScore()`` y comprobamos que se lanza una excepción mediante el método ``expectException()`` de PHPUnit.

Ejecuta las pruebas para comprobar que se han superado:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Añadamos pruebas para el *happy path* (probaremos las respuestas que puede generar la API sin tener en cuenta los eventos excepcionales que puedan suceder):

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

Los proveedores de datos de PHPUnit nos permiten reutilizar la misma lógica de prueba para varios casos de prueba.

Escribiendo pruebas funcionales para controladores
--------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Probar controladores es un poco diferente a probar una clase "normal" de PHP, ya que queremos ejecutarlos en el contexto de una petición HTTP.

Creando una prueba funcional para el controlador Conference:

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

El uso de ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` en lugar de ``PHPUnit\Framework\TestCase`` como clase base para nuestros tests nos brinda una buena abstracción para las pruebas funcionales.

La variable ``$client`` simula un navegador. En lugar de hacer llamadas HTTP al servidor, llama directamente a la aplicación Symfony. Esta estrategia tiene varios beneficios: es mucho más rápida que tener viajes de ida y vuelta entre el cliente y el servidor, pero también permite que las pruebas puedan inspeccionar el estado de los servicios después de cada petición HTTP.

Esta primera prueba comprueba que la página de inicio devuelve una respuesta HTTP 200.

Con el fin de facilitarnos la vida, a PHPUnit se le han incorporado comprobaciones (*asserts*) del tipo ``assertResponseIsSuccessful`` (comprobar si la respuesta es exitosa). Existen muchas de estas comprobaciones definidas por Symfony.

.. tip::

    Hemos utilizado la URL ``/`` en lugar de generarla a través del enrutador. Esto se hace a propósito ya que probar las URLs de los usuarios finales es parte de lo que queremos probar. Si cambias la ruta, las pruebas fallarán para recordarte que, probablemente, deberías redirigir la URL antigua a la nueva para no entorpecer a los motores de búsqueda y los sitios web que enlazan con tu sitio web.

.. note::

    Podríamos haber generado la prueba a través de maker:

    .. code-block:: bash

        $ symfony console make:test WebTestCase Controller\\ConferenceController

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
        </php>
    </phpunit>

.. index:: Command;secrets:set

Para hacer funcionar las pruebas, debemos definir la ``AKISMET_KEY``secreta para este entorno ``test``:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

.. note::

    Como hemos visto en un capítulo anterior, ``APP_ENV=test`` significa que la variable de entorno ``APP_ENV`` está configurada para el contexto del comando. En Windows, utiliza ``--env=test`` en su lugar: ``symfony console secrets:set AKISMET_KEY --env=test``

Trabajando con una base de datos de prueba
------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

Como ya hemos visto, Symfony CLI expone automaticamente la variable de entorno ``DATABASE_URL``. Cuando ``APP_ENV`` es ``test``, establecido al ejecutar PHPUnit, cambia el nombre de la base de datos ``main`` a ``main_test`` para que las pruebas tengan su propia base de datos. Esto es muy importante ya que necesitaremos algunos datos estables para ejecutar nuestras pruebas y no queremos sobreescribir lo almacenado en la base de datos de desarrollo.

Antes de poder ejecutar la prueba, necesitamos "inicializar" la base de datos de ``test`` (crear la base de datos y migrarla):

.. code-block:: bash

    $ APP_ENV=test symfony console doctrine:database:create
    $ APP_ENV=test symfony console doctrine:migrations:migrate -n

Si ahora ejecutas las pruebas, PHPUnit ya no interactuará con tu base de datos de desarrollo. Para ejecutar solo las nuevas pruebas, indica la ruta de sus clases:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Fíjate que hemos establecido ``APP_ENV`` de manera explícita incluso al ejecutar PHPUnit para permitir que Symfony CLI establezca el nombre de la base de datos como ``main_test``.

.. tip::

    Cuando una prueba falla, puede ser útil una introspección del objeto ``Response``. Accede a él a través de ``$client->getResponse()`` y ``echo`` para ver su aspecto.

Definiendo *fixtures*
---------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Para poder probar la lista de comentarios, la paginación y el envío del formulario, necesitamos que la base de datos contenga algunos datos. Y queremos que los datos sean los mismos entre prueba y prueba para que se pueda comprobar si pasan con éxito. Los *fixtures* son exactamente lo que necesitamos.

Instala el *bundle* Doctrine Fixtures:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Durante la instalación se ha creado un nuevo directorio ``src/DataFixtures/`` con una clase de ejemplo, lista para ser personalizada. Añade dos conferencias y un comentario por ahora:

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

Cuando carguemos los *fixtures*, se eliminarán todos los datos, incluido el usuario administrador. Para evitar eso, agreguemos el usuario admin a los *fixtures*:

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

    Si no recuerdas qué servicio se necesita utilizar para una tarea determinada, utiliza la opción ``debug:autowiring`` con alguna palabra clave:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Cargando *fixtures*
-------------------

.. index:: ! Command;doctrine:fixtures:load

Carga los *fixtures* para el entorno/base de datos ``test``:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load

Rastreo de un sitio web en pruebas funcionales
----------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Como hemos visto, el cliente HTTP utilizado en las pruebas simula un navegador, por lo que podemos navegar por la web como si estuviéramos utilizando un navegador tradicional.

Agrega una nueva prueba que haga clic en una página de la conferencia desde la página principal:

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

Describamos lo que sucede en esta prueba en un lenguaje sencillo:

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

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Enviando un formulario en una prueba funcional
----------------------------------------------

¿Quieres pasar al siguiente nivel? Inténtalo añadiendo un nuevo comentario con una foto en una conferencia desde una prueba simulando el envío de un formulario. Eso parece ambicioso, ¿no? Mira el código necesario: no es más complejo que el que ya hemos escrito:

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

Para enviar un formulario a través de ``submitForm()``, busca los nombres de los *inputs* gracias al navegador DevTools o a través del panel del Symfony Profiler Form. ¡Observa la elegante reutilización de la imagen en construcción!

Vuelve a realizar las pruebas para comprobar que todo está en verde:

.. code-block:: bash

    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Si quieres comprobar el resultado en un navegador, para el servidor web y vuelve a ejecutarlo en el entorno ``test``:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Recargando los *fixtures*
-------------------------

.. index::
    single: Command;doctrine:fixtures:load

Si haces las pruebas por segunda vez, deberían fallar. Como ahora hay más comentarios en la base de datos, la comprobación que verifica el número de comentarios fallará. Necesitamos restablecer el estado de la base de datos entre cada ejecución, recargando los *fixtures* antes de cada ejecución:

.. code-block:: bash
    :class: answers(y)

    $ APP_ENV=test symfony console doctrine:fixtures:load
    $ APP_ENV=test symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatizando el flujo de trabajo con un Makefile
-------------------------------------------------

.. index::
    single: Makefile

Tener que recordar la secuencia de comandos que ejecuta las pruebas es molesto. Debería, al menos, estar documentado. Pero la documentación debe ser el último recurso. En cambio, ¿qué hay de la automatización de las actividades cotidianas? Eso serviría como documentación, ayudaría a otros desarrolladores a descubrirlo y les haría la vida más fácil y rápida.

.. index::
    single: Command;doctrine:fixtures:load

Usar un ``Makefile`` es una forma de automatizar comandos:

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

    En una regla Makefile, la sangría **debe** consistir en un único carácter de tabulación en lugar de espacios.

Observa el parámetro ``-n`` en el comando Doctrine; es un parámetro global de los comandos Symfony que los hace no interactivos.

Siempre que desees ejecutar las pruebas, utiliza ``make tests``:

.. code-block:: bash

    $ make tests

Restableciendo la base de datos después de cada prueba
-------------------------------------------------------

.. index::
    single: PHPUnit;Performance

Reiniciar la base de datos después de cada prueba es bueno, pero tener pruebas verdaderamente independientes es aún mejor. No queremos que una prueba se base en los resultados de las anteriores. Cambiar el orden de las pruebas no debe cambiar el resultado. Como vamos a descubrir ahora, éste no es el caso por el momento.

Mueve la prueba ``testConferencePage`` después de ``testCommentSubmission``:

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

Las pruebas ahora fallan.

.. index::
    single: Doctrine;TestBundle

Para restablecer la base de datos entre pruebas, instala DoctrineTestBundle:

.. code-block:: bash
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: bash

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

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

Habilita el oyente de PHPUnit:

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

Y hecho. Cualquier cambio realizado en las pruebas se retrotrae automáticamente al final de cada prueba.

Las pruebas deberían de nuevo estar en verde:

.. code-block:: bash

    $ make tests

Usando un navegador real para pruebas funcionales
-------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Las pruebas funcionales utilizan un navegador especial que llama directamente a la capa de Symfony. Pero también puedes usar un navegador real y la capa HTTP real gracias a Symfony Panther:

.. code-block:: bash

    $ symfony composer req panther --dev

Puedes escribir pruebas que utilicen un navegador Google Chrome real con los siguientes cambios:

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

La variable de entorno ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contiene la URL del servidor web local.

Ejecutando pruebas funcionales de caja negra (*Black Box*) con Blackfire
------------------------------------------------------------------------

Otra forma de realizar pruebas funcionales es utilizar el `reproductor Blackfire <https://blackfire.io/player>`_. Además de lo que puedes hacer con las pruebas funcionales, también puedes realizar pruebas de rendimiento.

Consulta el paso "Rendimiento" para obtener más información.

.. sidebar:: Yendo más allá

    * `Lista de comprobaciones (*assertions*) definidas por Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ para pruebas funcionales;

    * `Documentación de PHPUnit <https://phpunit.de/documentation.html>`_;

    * La `biblioteca Faker <https://github.com/FakerPHP/Faker>`_ para generar datos realistas de *fixtures*;

    * La documentación del `componente CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * La biblioteca `Symfony Panther <https://github.com/symfony/panther>`_ para pruebas de navegadores y rastreo web en aplicaciones Symfony;

    * La `documentación Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
