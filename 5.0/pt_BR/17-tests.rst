Testes
======

.. index::
    single: PHPUnit

Já que começamos a adicionar mais e mais funcionalidades na aplicação, agora provavelmente é o momento certo para falar sobre testes.

*Fato engraçado*: Encontrei um bug enquanto escrevia os testes neste capítulo.

O Symfony depende do PHPUnit para testes unitários. Vamos instalá-lo:

.. code-block:: bash

    $ symfony composer req phpunit --dev

Escrevendo Testes Unitários
----------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:unit-test

``SpamChecker`` é a primeira classe para a qual vamos escrever testes. Gere um teste unitário:

.. code-block:: bash

    $ symfony console make:unit-test SpamCheckerTest

Testar o SpamChecker é um desafio, pois certamente não queremos acessar a API do Akismet. Nós vamos *simular* a API.

.. index::
    single: Mock

Vamos escrever um primeiro teste para quando a API retorna um erro:

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

A classe ``MockHttpClient`` torna possível simular qualquer servidor HTTP. Ela recebe um array de instâncias ``MockResponse`` que contêm o corpo esperado e os cabeçalhos do Response.

Então, chamamos o método ``getSpamScore()`` e verificamos se uma exceção é lançada através do método ``expectException()`` do PHPUnit.

Execute os testes para verificar se eles passam:

.. code-block:: bash

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Annotations;@dataProvider

Vamos adicionar testes para o caminho feliz:

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

Os provedores de dados do PHPUnit nos permitem reutilizar a mesma lógica de teste para vários casos de teste.

Escrevendo Testes Funcionais para Controladores
-----------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit
    single: Command;make:functional-test

Testar os controladores é um pouco diferente de testar uma classe "regular" do PHP, pois queremos executá-los no contexto de uma requisição HTTP.

Install some extra dependencies needed for functional tests:

.. code-block:: bash

    $ symfony composer req browser-kit --dev

Crie um teste funcional para o controlador da conferência:

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

Este primeiro teste verifica se a página inicial retorna uma resposta HTTP 200.

A variável ``$client`` simula um navegador. Em vez de fazer chamadas HTTP para o servidor, ela chama a aplicação Symfony diretamente. Essa estratégia tem vários benefícios: é muito mais rápido do que ter várias idas e vindas entre o cliente e o servidor, mas também permite que os testes examinem o estado dos serviços após cada requisição HTTP.

Asserções como ``assertResponseIsSuccessful`` são adicionadas ao PHPUnit para facilitar o seu trabalho. Há muitas dessas asserções definidas pelo Symfony.

.. tip::

    Usamos ``/`` como URL ao invés de gerá-la através do roteador. Isso é feito de propósito, pois testar URLs de usuários finais faz parte do que queremos testar. Se você alterar o caminho da rota, os testes irão quebrar como um bom lembrete de que você provavelmente deve redirecionar a antiga URL para a nova para ser agradável com os motores de busca e sites com links para o seu site.

.. note::

    Podíamos ter gerado o teste através do bundle Maker:

    .. code-block:: bash

        $ symfony console make:functional-test Controller\\ConferenceController

.. index:: Command;secrets:set

Os testes do PHPUnit são executados em um ambiente ``test`` dedicado. Devemos definir o valor do segredo ``AKISMET_KEY`` para esse ambiente:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ APP_ENV=test symfony console secrets:set AKISMET_KEY

Run the new tests only by passing the path to their class:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    Quando um teste falha, pode ser útil examinar o objeto Response. Acesse-o via ``$client->getResponse()`` e use ``echo`` nele para ver seu conteúdo.

Definindo Fixtures
------------------

.. index::
    single: Doctrine;Fixtures
    single: Fixtures

Para poder testar a lista de comentários, a paginação e a submissão de formulários, precisamos popular o banco de dados com alguns dados. E queremos que os dados sejam estáveis entre as execuções dos testes para que eles passem. As fixtures são exatamente o que precisamos.

Instale o bundle Doctrine Fixtures:

.. code-block:: bash

    $ symfony composer req orm-fixtures --dev

Um novo diretório ``src/DataFixtures/`` foi criado durante a instalação com uma classe de amostra, pronta para ser personalizada. Adicione duas conferências e um comentário por enquanto:

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

Quando carregarmos as fixtures, todos os dados serão removidos, incluindo o usuário admin. Para evitar isso, vamos adicionar o usuário admin nas fixtures:

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

    Se você não lembra qual serviço precisa usar para uma determinada tarefa, use o ``debug:autowiring`` com alguma palavra-chave:

    .. code-block:: bash

        $ symfony console debug:autowiring encoder

Carregando as Fixtures
----------------------

.. index:: ! Command;doctrine:fixtures:load

Load the fixtures into the database. **Be warned** that it will delete *all* data currently stored in the database (if you want to avoid this behavior, keep reading).

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load

Coletando Dados de um Site em Testes Funcionais
-----------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

Como vimos, o cliente HTTP usado nos testes simula um navegador, então podemos navegar pelo site como se estivéssemos usando um navegador sem interface.

Adicione um novo teste que clica em uma página de conferência a partir da página inicial:

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

Vamos descrever em inglês o que acontece neste teste:

* Como no primeiro teste, vamos para a página inicial;

* O método ``request()`` retorna uma instância ``Crawler`` que ajuda a encontrar elementos na página (como links, formulários ou qualquer coisa que você possa obter com seletores CSS ou XPath);

* Graças a um seletor CSS, confirmamos que temos duas conferências listadas na página inicial;

* Em seguida, clicamos no link "View" (como ele não pode clicar em mais de um link por vez, o Symfony escolhe automaticamente o primeiro que encontrar);

* Confirmamos o título da página, a resposta e o título ``<h2>`` da página para ter certeza de que estamos na página certa (também poderíamos ter verificado a rota correspondente);

* Finalmente, confirmamos que há 1 comentário na página. ``div:contains()`` não é um seletor CSS válido, mas o Symfony tem algumas adições agradáveis, emprestadas do jQuery.

Em vez de clicar no texto (ou seja, ``View``), poderíamos ter selecionado o link através de um seletor CSS também:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

Verifique se o novo teste está verde:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Trabalhando com um Banco de Dados de Teste
------------------------------------------

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

Carregue as fixtures para o ambiente/banco de dados ``test``:

.. code-block:: bash
    :class: ignore

    $ APP_ENV=test symfony console doctrine:fixtures:load

For the rest of this step, we won't redefine the ``DATABASE_URL`` environment variable. Using the same database as the ``dev`` environment for tests has some advantages we will see in the next section.

Submetendo um Formulário em um Teste Funcional
-----------------------------------------------

Quer ir para o próximo nível? Tente adicionar um novo comentário com uma foto em uma conferência a partir de um teste simulando a submissão de um formulário. Parece ambicioso, não parece? Olhe o código necessário: não é mais complexo do que o que já escrevemos:

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

Para enviar um formulário via ``submitForm()``, encontre os nomes dos campos usando as ferramentas de desenvolvedor do navegador ou através do painel Form do Profiler do Symfony. Observe a reutilização inteligente da imagem em construção!

Execute os testes novamente para verificar se está tudo verde:

.. code-block:: bash

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Uma vantagem de usar o banco de dados "dev" para os testes é que você pode verificar o resultado em um navegador:

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Recarregando as Fixtures
------------------------

.. index::
    single: Command;doctrine:fixtures:load

Se você executar os testes uma segunda vez, eles devem falhar. Como existem agora mais comentários no banco de dados, a assertion que verifica o número de comentários está quebrada. Precisamos resetar o estado do banco de dados entre cada execução recarregando as fixtures antes de cada execução:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:fixtures:load
    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

Automatizando seu Fluxo de Trabalho com um Makefile
---------------------------------------------------

.. index::
    single: Makefile

Ter que lembrar uma sequência de comandos para executar os testes é irritante. Isso deve, pelo menos, ser documentado. Mas a documentação deve ser um último recurso. Em vez disso, que tal automatizar as atividades diárias? Isso serviria como documentação, ajudaria na descoberta por outros desenvolvedores e tornaria a vida dos desenvolvedores mais fácil e rápida.

.. index::
    single: Command;doctrine:fixtures:load

Usar um ``Makefile`` é uma forma de automatizar comandos:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:fixtures:load -n
    	symfony php bin/phpunit
    .PHONY: tests

Observe a flag ``-n`` no comando do Doctrine; é uma flag global nos comandos do Symfony que os torna não interativos.

Sempre que você quiser executar os testes, use `make tests``:

.. code-block:: bash

    $ make tests

Resetando o Banco de Dados após Cada Teste
-------------------------------------------

.. index::
    single: PHPUnit;Performance

Resetar o banco de dados após cada execução de teste é bom, mas ter testes verdadeiramente independentes é ainda melhor. Não queremos que um teste conte com os resultados dos anteriores. Alterar a ordem dos testes não deve alterar o resultado. Como vamos descobrir agora, esse não é o caso no momento.

Mova o teste ``testConferencePage`` para depois do ``testCommentSubmission``:

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

Os testes agora falham.

.. index::
    single: Doctrine;TestBundle

Para resetar o banco de dados entre os testes, instale o DoctrineTestBundle:

.. code-block:: bash
    :class: answers(p)

    $ symfony composer req "dama/doctrine-test-bundle:^6" --dev

Você precisará confirmar a execução da receita (já que não é um bundle suportado "oficialmente"):

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

Ative o listener do PHPUnit:

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

E pronto. Quaisquer alterações feitas nos testes agora são automaticamente revertidas no final de cada teste.

Os testes devem estar verdes novamente:

.. code-block:: bash

    $ make tests

Usando um Navegador Real para Testes Funcionais
-----------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

Os testes funcionais usam um navegador especial que chama a camada Symfony diretamente. Mas, você também pode usar um navegador real e a camada HTTP real, graças ao Panther do Symfony:

.. code-block:: bash

    $ symfony composer req panther --dev

Você pode então escrever testes que utilizem um navegador Google Chrome real com as seguintes alterações:

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

A variável de ambiente ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` contém a URL do servidor web local.

Executando Testes Funcionais Caixa Preta com Blackfire
------------------------------------------------------

Outra forma de realizar testes funcionais é usar o `player do Blackfire <https://blackfire.io/player>`_. Além do que você pode fazer com testes funcionais, ele também pode realizar testes de desempenho.

Consulte a etapa sobre "Desempenho" para saber mais.

.. sidebar:: Indo Além

    * `Lista de assertions definidas pelo Symfony <https://symfony.com/doc/current/testing/functional_tests_assertions.html>`_ para testes funcionais;

    * `Documentação do PHPUnit <https://phpunit.de/documentation.html>`_;

    * A `biblioteca Faker <https://github.com/fzaninotto/Faker>`_ para gerar fixtures com dados realistas;

    * A `documentação do componente CssSelector <https://symfony.com/doc/current/components/css_selector.html>`_;

    * A biblioteca `Panther do Symfony <https://github.com/symfony/panther>`_ para testes de navegador e coleta de dados da web em aplicações Symfony;

    * A `documentação do Make/Makefile <https://www.gnu.org/software/make/manual/make.html>`_.
