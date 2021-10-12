Armazenando em Cache para Desempenho
====================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Problemas de desempenho podem vir com a popularidade. Alguns exemplos típicos: ausência de índices no banco de dados ou excesso de requisições SQL por página. Você não terá problemas com um banco de dados vazio, porém eles podem surgir em algum momento devido ao aumento de tráfego e volume de dados.

Adicionando Cabeçalhos de Cache HTTP
-------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Usar estratégias de cache HTTP é uma ótima maneira de maximizar o desempenho para usuários finais com pouco esforço. Adicione um cache de proxy reverso em produção para habilitar o armazenamento em cache, e use uma `CDN`_ para armazenar em cache na ponta para um desempenho ainda melhor.

Vamos armazenar a página inicial em cache por uma hora:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -35,9 +35,12 @@ class ConferenceController extends AbstractController
          */
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/index.html.twig', [
    +        $response = new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         /**

O método ``setSharedMaxAge()`` configura a expiração do cache para proxies reversos. Use o método ``setMaxAge()`` para controlar o cache do navegador. O tempo é representado em segundos (1 hora = 60 minutos = 3600 segundos).

O cache da página da conferência é mais desafiador, pois é mais dinâmico. Qualquer um pode adicionar um comentário a qualquer momento, e ninguém quer esperar uma hora para vê-lo online. Nesses casos, use a estratégia de *validação HTTP*.

Ativando o Kernel de Cache HTTP do Symfony
------------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Para testar a estratégia de cache HTTP, use o proxy reverso HTTP do Symfony:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -1,6 +1,7 @@
     <?php

     use App\Kernel;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\ErrorHandler\Debug;
     use Symfony\Component\HttpFoundation\Request;

    @@ -21,6 +22,11 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? false) {
     }

     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
    +
    +if ('dev' === $kernel->getEnvironment()) {
    +    $kernel = new HttpCache($kernel);
    +}
    +
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);

Além de ser um proxy reverso HTTP completo, o proxy reverso HTTP do Symfony (através da classe ``HttpCache``) adiciona algumas informações de depuração como cabeçalhos HTTP. Isso ajuda muito na validação dos cabeçalhos do cache que definimos.

Confira na página inicial:

.. code-block:: bash
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

Para a primeira requisição, o servidor de cache diz que foi um ``miss`` e que ele executou um ``store`` para armazenar a resposta em cache. Verifique o cabeçalho ``cache-control`` para ver a estratégia de cache configurada.

Para requisições subsequentes, a resposta já estará no cache (o cabeçalho ``age`` também foi atualizado):

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

Evitando Requisições SQL com ESI
----------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

O listener ``TwigEventSubscriber`` injeta uma variável global no Twig com todos os objetos da conferência. Ele faz isso para cada página do site. É provavelmente um ótimo alvo para otimização.

Você não vai adicionar novas conferências todos os dias, então o código está consultando os mesmos dados do banco de dados repetidamente.

Nós podemos querer armazenar em cache os nomes e slugs da conferência com o componente Cache do Symfony, mas sempre que possível eu confio na infraestrutura de cache do HTTP.

Quando você quiser armazenar em cache um fragmento de uma página, mova-o para fora da requisição HTTP atual, criando uma *sub-requisição*. *ESI* é uma combinação perfeita para este caso de uso. Um ESI é uma maneira de incorporar o resultado de uma solicitação HTTP em outra.

Crie um controlador que retorne somente o fragmento HTML que exibe as conferências:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -45,6 +45,16 @@ class ConferenceController extends AbstractController
             return $response;
         }

    +    /**
    +     * @Route("/conference_header", name="conference_header")
    +     */
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return new Response($this->twig->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
    +    }
    +
         /**
          * @Route("/conference/{slug}", name="conference")
          */

Crie o template correspondente:

.. code-block:: twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Acesse ``/conference_header`` para verificar se tudo está funcionando bem.

.. index::
    single: Twig;render
    single: Twig;path

Hora de revelar o truque! Atualize o layout do Twig para chamar o controlador que acabamos de criar:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,11 +8,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

E *voilà*. Atualize a página e o site ainda está exibindo a mesma coisa.

.. tip::

    Use o painel "Request / Response" do Profiler do Symfony para saber mais sobre a requisição principal e suas sub-requisições.

Agora, cada vez que você acessa uma página no navegador, duas requisições HTTP são executadas, uma para o cabeçalho e outra para a página principal. Você piorou o desempenho. Parabéns!

A chamada HTTP do cabeçalho da conferência é atualmente feita internamente pelo Symfony, portanto, não envolve idas e vindas entre cliente e servidor. Isso também significa que não há nenhuma maneira de se beneficiar dos cabeçalhos de cache HTTP.

Converta a chamada em uma chamada HTTP "real" usando um ESI.

Primeiro, habilite o suporte a ESI:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -11,7 +11,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Então, use ``render_esi`` ao invés de ``render``:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -8,7 +8,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Se o Symfony detectar um proxy reverso que seja capaz de lidar com ESI, ele habilita o suporte automaticamente (caso contrário, a renderização da sub-requisição será realizada de forma síncrona).

Como o proxy reverso do Symfony suporta ESIs, vamos verificar seus logs (limpe o cache primeiro - veja como abaixo):

.. code-block:: bash
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

Atualize algumas vezes: a resposta de ``/`` está em cache e a de ``/conference_header`` não. Conseguimos algo fantástico: ter a página inteira no cache e ainda assim ter uma parte dinâmica.

Mas não é isto que queremos. Faça cache do cabeçalho da página por uma hora, independentemente de todo o resto:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -48,9 +48,12 @@ class ConferenceController extends AbstractController
          */
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/header.html.twig', [
    +        $response = new Response($this->twig->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         /**

O cache agora está habilitado para ambas as requisições:

.. code-block:: bash
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

O cabeçalho ``x-symfony-cache`` contém dois elementos: a requisição principal ``/`` e uma sub-requisição (o ESI do ``conference_header``). Ambos estão no cache (``fresh``).

As estratégias de cache da página principal e de seus ESIs podem ser diferentes. Se tivermos uma página "about", podemos querer armazená-la em cache por uma semana, e ainda assim ter o cabeçalho atualizado de hora em hora.

Remova o listener, pois já não precisamos dele:

.. code-block:: bash

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Limpando o Cache HTTP para Testes
---------------------------------

Testar o site em um navegador ou através de testes automatizados torna-se um pouco mais difícil com uma camada de cache.

You can manually remove all the HTTP cache by removing the
``var/cache/dev/http_cache/`` directory:

.. code-block:: bash

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Annotations;@Route

Essa estratégia não funciona bem se você quiser invalidar apenas algumas URLs, ou se você quiser integrar a invalidação do cache em seus testes funcionais. Vamos adicionar um pequeno endpoint HTTP, apenas para administração, para invalidar algumas URLs:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -6,8 +6,10 @@ use App\Entity\Comment;
     use App\Message\CommentMessage;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
    @@ -54,4 +56,19 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]);
         }
    +
    +    /**
    +     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     */
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store = (new class($kernel) extends HttpCache {})->getStore();
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

O novo controlador foi restringido ao método HTTP ``PURGE``. Este método não está no padrão HTTP, mas é amplamente utilizado para invalidar caches.

Por padrão, os parâmetros da rota não podem conter ``/``, pois isso separa os segmentos de URL. Você pode substituir esta restrição para o último parâmetro da rota, por exemplo ``uri``, definindo o seu próprio padrão de requisitos (``.*``).

A forma como obtemos a instância de ``HttpCache`` também pode parecer um pouco estranha; estamos usando uma classe anônima, pois não é possível acessar a classe "real". A instância de ``HttpCache``  encapsula o kernel real, que desconhece a camada de cache, como deve ser.

Invalide a página inicial e o cabeçalho da página de conferência através das seguintes chamadas cURL:

.. code-block:: bash

    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

O subcomando ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` retorna a URL atual do servidor web local.

.. note::

    O controlador não tem um nome de rota porque ela nunca será referenciada no código.

Agrupando Rotas Similares com um Prefixo
----------------------------------------

.. index::
    single: Annotations;@Route

As duas rotas no controlador de administração têm o mesmo prefixo ``/admin``. Em vez de repeti-lo em todas as rotas, refatore-as configurando o prefixo na própria classe:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,9 @@ use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    +/**
    + * @Route("/admin")
    + */
     class AdminController extends AbstractController
     {
         private $twig;
    @@ -29,7 +32,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/comment/review/{id}", name="review_comment")
    +     * @Route("/comment/review/{id}", name="review_comment")
          */
         public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
         {
    @@ -58,7 +61,7 @@ class AdminController extends AbstractController
         }

         /**
    -     * @Route("/admin/http-cache/{uri<.*>}", methods={"PURGE"})
    +     * @Route("/http-cache/{uri<.*>}", methods={"PURGE"})
          */
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
         {

Armazenando Operações Intensivas de CPU/Memória em Cache
-----------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

O site não possui algoritmos que demandem muito processamento ou memória. Para falar sobre *caches locais*, vamos criar um comando que mostra o passo atual em que estamos trabalhando (mais precisamente o nome da tag atrelada ao commit atual).

O componente Process do Symfony permite que você execute um comando e recupere o resultado (saídas padrão e de erro); instale-o:

.. code-block:: bash

    $ symfony composer req process

Implemente o comando:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    class StepInfoCommand extends Command
    {
        protected static $defaultName = 'app:step:info';

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return 0;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Você pode utilizar ``make:command`` para criar o comando:

    .. code-block:: bash
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

E se quisermos armazenar a saída em cache por alguns minutos? Use o Cache do Symfony:

.. code-block:: bash

    $ symfony composer req cache

E envolva o código com a lógica do cache:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
    @@ -6,16 +6,31 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     class StepInfoCommand extends Command
     {
         protected static $defaultName = 'app:step:info';

    +    private $cache;
    +
    +    public function __construct(CacheInterface $cache)
    +    {
    +        $this->cache = $cache;
    +
    +        parent::__construct();
    +    }
    +
         protected function execute(InputInterface $input, OutputInterface $output): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $this->cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return 0;
         }

O processo só é chamado agora se o item ``app.current_step`` não estiver no cache.

Criando Profiles e Comparando o Desempenho
------------------------------------------

Nunca adicione cache cegamente. Tenha em mente que adicionar algum cache adiciona uma camada de complexidade. E como somos todos muito ruins em adivinhar o que será rápido e o que é lento, você pode acabar em uma situação em que o cache torna sua aplicação mais lenta.

Sempre meça o impacto da adição de um cache com um profiler como o `Blackfire <https://blackfire.io/>`_.

Consulte a etapa sobre "Desempenho" para saber mais sobre como você pode usar o Blackfire para testar seu código antes da implantação.

Configurando um Cache de Proxy Reverso em Produção
----------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: SymfonyCloud;Varnish
    single: Varnish

Não utilize o proxy reverso do Symfony em produção. Sempre prefira um proxy reverso como o Varnish em sua infraestrutura ou uma CDN comercial.

Adicione o Varnish aos serviços da SymfonyCloud:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -10,3 +10,12 @@ queue:
         type: rabbitmq:3.7
         disk: 1024
         size: S
    +
    +varnish:
    +    type: varnish:6.0
    +    relationships:
    +        application: 'app:http'
    +    configuration:
    +        vcl: !include
    +            type: string
    +            path: config.vcl

.. index::
    single: SymfonyCloud;Routes

Utilize o Varnish como ponto de entrada principal nas rotas:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Finalmente, crie um arquivo ``config.vcl`` para configurar o Varnish:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Habilitando o Suporte a ESI no Varnish
--------------------------------------

O suporte a ESI no Varnish deve ser habilitado explicitamente para cada requisição. Para torná-lo universal, o Symfony usa os cabeçalhos padrão ``Surrogate-Capability`` e ``Surrogate-Control`` para negociar o suporte a ESI:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Limpando o Cache do Varnish
---------------------------

Invalidar o cache em produção provavelmente nunca deveria ser necessário, exceto em situações de emergência e talvez em branches que não sejam o ``master``. Se você precisar limpar o cache frequentemente, isso provavelmente significa que a estratégia de cache deve ser ajustada (diminuindo o TTL ou usando uma estratégia de validação ao invés de uma de expiração).

De qualquer forma, vamos ver como configurar o Varnish para a invalidação do cache:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

Na vida real, você provavelmente restringiria por IPs ao invés disso, como descrito na `documentação do Varnish <https://varnish-cache.org/docs/trunk/users-guide/purging.html>`_.

Limpe algumas URLs agora:

.. code-block:: bash

    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`
    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`conference_header

As URLs parecem um pouco estranhas porque as que são retornadas por ``env:urls`` já terminam com ``/``.

.. sidebar:: Indo Além

    * `Cloudflare <https://www.cloudflare.com>`_, a plataforma de nuvem global;

    * `Documentação do Varnish HTTP Cache <https://varnish-cache.org/docs/index.html>`_;

    * `Especificação ESI <https://www.w3.org/TR/esi-lang>`_ e `recursos do desenvolvedor ESI <https://www.akamai.com/us/en/support/esi.jsp>`_;

    * `Modelo de validação de cache HTTP <https://symfony.com/doc/current/http_cache/validation.html>`_;

    * `HTTP Cache in SymfonyCloud <https://symfony.com/doc/current/cloud/cookbooks/cache.html>`_.

.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
