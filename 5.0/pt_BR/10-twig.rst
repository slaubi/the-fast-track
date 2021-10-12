Construindo a Interface de Usuário
===================================

.. index::
    single: Twig
    single: Templates

Agora tudo está pronto para criar a primeira versão da interface de usuário do site. Não vamos torná-la bonita. Apenas funcional por enquanto.

Lembra do escape que precisamos fazer no controlador para evitar problemas de segurança com o easter egg? Não vamos usar PHP nos nossos templates por essa razão. Em vez disso, vamos usar o Twig. Além de lidar com escape de saída para nós, o `Twig`_ traz muitos recursos agradáveis que vamos aproveitar, como herança de template.

Instalando o Twig
-----------------

Não precisamos adicionar o Twig como uma dependência, pois ele já foi instalado como uma *dependência transitiva* do EasyAdmin. Mas, e se no futuro você decidir mudar para outro bundle de administração? Um que use uma API e um frontend em React, por exemplo. Ele provavelmente já não dependerá mais do Twig, que será automaticamente removido quando você remover o EasyAdmin.

Por precaução, vamos dizer ao Composer que o projeto realmente depende do Twig, independentemente do EasyAdmin. Adicioná-lo como qualquer outra dependência é suficiente:

.. code-block:: bash

    $ symfony composer req twig

O Twig agora faz parte das principais dependências do projeto no ``composer.json``:

.. code-block:: diff
    :class: ignore

    --- a/composer.json
    +++ b/composer.json
    @@ -14,6 +14,7 @@
             "symfony/framework-bundle": "4.4.*",
             "symfony/maker-bundle": "^1.0@dev",
             "symfony/orm-pack": "dev-master",
    +        "symfony/twig-pack": "^1.0",
             "symfony/yaml": "4.4.*"
         },
         "require-dev": {

Usando o Twig para os Templates
-------------------------------

.. index::
    single: Twig;Layout
    single: Twig;block

Todas as páginas do site compartilharão o mesmo *layout*. Durante a instalação do Twig um diretório ``templates/`` foi criado automaticamente e um layout de exemplo também foi criado em ``base.html.twig``.

.. code-block:: twig
    :caption: templates/base.html.twig
    :class: ignore

    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            {% block stylesheets %}{% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
            {% block javascripts %}{% endblock %}
        </body>
    </html>

Um layout pode definir elementos ``block``, que são os locais onde os *templates filhos* que *estendem* o layout adicionam seus conteúdos.

.. index::
    single: Twig;extends
    single: Twig;for

Vamos criar um template para a página inicial do projeto em ``templates/conference/index.html.twig``:

.. code-block:: twig
    :caption: templates/conference/index.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook{% endblock %}

    {% block body %}
        <h2>Give your feedback!</h2>

        {% for conference in conferences %}
            <h4>{{ conference }}</h4>
        {% endfor %}
    {% endblock %}

O template *estende* ``base.html.twig`` e redefine os blocos ``title`` e ``body``.

.. index::
    single: Twig;Syntax

A notação ``{% %}`` em um template indica *ações* e *estruturas*.

A notação ``{{ }}`` é usada para *exibir* algo. ``{{ conference }}`` exibe a representação da conferência (o resultado da chamada a ``__toString`` no objeto ``Conference``).

Usando o Twig em um Controlador
-------------------------------

Atualize o controlador para renderizar o template Twig:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,24 +2,21 @@

     namespace App\Controller;

    +use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
    +use Twig\Environment;

     class ConferenceController extends AbstractController
     {
         /**
          * @Route("/", name="homepage")
          */
    -    public function index(): Response
    +    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response(<<<EOF
    -<html>
    -    <body>
    -        <img src="/images/under-construction.gif" />
    -    </body>
    -</html>
    -EOF
    -        );
    +        return new Response($twig->render('conference/index.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
         }
     }

Há muita coisa acontecendo aqui.

Para podermos renderizar um template, precisamos do objeto ``Environment`` do Twig (o ponto de entrada principal do Twig). Repare que pedimos a instância do Twig declarando seu tipo no método do controlador. O Symfony é inteligente o suficiente para saber como injetar o objeto certo.

Também precisamos do repositório da conferência para obter todas as conferências do banco de dados.

No código do controlador, o método ``render()`` renderiza o template e passa um array de variáveis para o template. Estamos passando a lista de objetos ``Conference`` como uma variável ``conferences``.

Um controlador é uma classe PHP padrão. Nós não precisamos nem mesmo estender a classe ``AbstractController`` se quisermos ser explícitos sobre nossas dependências. Você pode removê-la (mas não faça isso, pois usaremos os bons atalhos que ela fornece em etapas futuras).

Criando a Página para uma Conferência
---------------------------------------

Cada conferência deve ter uma página própria para listar seus comentários. Adicionar uma nova página é uma questão de adicionar um controlador, definir uma rota para ele e criar o template relacionado.

Adicione um método ``show()`` em ``src/Controller/ConferenceController.php``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,6 +2,8 @@

     namespace App\Controller;

    +use App\Entity\Conference;
    +use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    @@ -19,4 +21,15 @@ class ConferenceController extends AbstractController
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    +
    +    /**
    +     * @Route("/conference/{id}", name="conference")
    +     */
    +    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    {
    +        return new Response($twig->render('conference/show.html.twig', [
    +            'conference' => $conference,
    +            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +        ]));
    +    }
     }

Este método tem um comportamento especial que ainda não vimos. Pedimos que uma instância de ``Conference`` seja injetada no método. Mas pode haver muitos destes no banco de dados. O Symfony é capaz de determinar qual deles você quer com base no ``{id}`` passado no caminho da requisição (``id`` sendo a chave primária da tabela ``conference`` no banco de dados).

A busca dos comentários relacionados à conferência pode ser feita através do método ``findBy()``, que recebe um critério como primeiro argumento.

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;for
    single: Twig;if
    single: Twig;else
    single: Twig;asset
    single: Twig;format_datetime
    single: Twig;length

O último passo é criar o arquivo ``templates/conference/show.html.twig``:

.. code-block:: twig
    :caption: templates/conference/show.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

    {% block body %}
        <h2>{{ conference }} Conference</h2>

        {% if comments|length > 0 %}
            {% for comment in comments %}
                {% if comment.photofilename %}
                    <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" />
                {% endif %}

                <h4>{{ comment.author }}</h4>
                <small>
                    {{ comment.createdAt|format_datetime('medium', 'short') }}
                </small>

                <p>{{ comment.text }}</p>
            {% endfor %}
        {% else %}
            <div>No comments have been posted yet for this conference.</div>
        {% endif %}
    {% endblock %}

Neste template, estamos usando a notação ``|`` para executar os *filtros* do Twig. Um filtro transforma um valor. ``comments|length`` retorna o número de comentários e ``comment.createdAt|format_datetime('medium', 'short')`` formata a data em uma representação humana legível.

Tente acessar a "primeira" conferência através de ``/conference/1``, e observe o seguinte erro:

.. figure:: screenshots/intl-twig-error.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

O erro vem do filtro ``format_datetime``, pois ele não faz parte do núcleo do Twig. A mensagem de erro fornece uma dica sobre qual pacote deve ser instalado para corrigir o problema:

.. code-block:: bash

    $ symfony composer req "twig/intl-extra:^3"

Agora a página funciona corretamente.

Interligando as Páginas
------------------------

.. index::
    single: Twig;Link
    single: Link

O último passo para terminar a nossa primeira versão da interface de usuário é adicionar um link para as páginas da conferência a partir da página inicial:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -7,5 +7,8 @@

         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
    +        <p>
    +            <a href="/conference/{{ conference.id }}">View</a>
    +        </p>
         {% endfor %}
     {% endblock %}

Mas adicionar um caminho fixo no código é uma má ideia por várias razões. O motivo mais importante é que se você modificar o caminho (de ``/conference/{id}`` para ``/conferences/{id}``, por exemplo), todos os links devem ser atualizados manualmente.

.. index::
    single: Twig;path

Em vez disso, utilize a *função* ``path()`` do Twig e utilize o *nome da rota*:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="/conference/{{ conference.id }}">View</a>
    +            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}

A função ``path()`` gera o caminho para uma página utilizando o nome da rota. Os valores dos parâmetros da rota são passados como um mapa do Twig.

Paginando os Comentários
-------------------------

.. index::
    single: Doctrine;Paginator
    single: Paginator

Com milhares de participantes, podemos esperar alguns comentários. Se exibirmos todos eles em uma única página, ela crescerá muito rápido.

Crie um método ``getCommentPaginator()`` no repositório de comentários que retorna um *Paginator* de comentários com base em uma conferência e um offset (por onde começar):

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -3,8 +3,10 @@
     namespace App\Repository;

     use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
      * @method Comment|null find($id, $lockMode = null, $lockVersion = null)
    @@ -14,11 +16,27 @@ use Doctrine\Persistence\ManagerRegistry;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    public const PAGINATOR_PER_PAGE = 2;
    +
         public function __construct(ManagerRegistry $registry)
         {
             parent::__construct($registry, Comment::class);
         }

    +    public function getCommentPaginator(Conference $conference, int $offset): Paginator
    +    {
    +        $query = $this->createQueryBuilder('c')
    +            ->andWhere('c.conference = :conference')
    +            ->setParameter('conference', $conference)
    +            ->orderBy('c.createdAt', 'DESC')
    +            ->setMaxResults(self::PAGINATOR_PER_PAGE)
    +            ->setFirstResult($offset)
    +            ->getQuery()
    +        ;
    +
    +        return new Paginator($query);
    +    }
    +
         // /**
         //  * @return Comment[] Returns an array of Comment objects
         //  */

Definimos o número máximo de comentários por página como 2 para facilitar os testes.

Para gerenciar a paginação no template, passe o Paginator do Doctrine em vez da Collection do Doctrine para o Twig:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -6,6 +6,7 @@ use App\Entity\Conference;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;
    @@ -25,11 +26,16 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{id}", name="conference")
          */
    -    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $offset = max(0, $request->query->getInt('offset', 0));
    +        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
    +
             return new Response($twig->render('conference/show.html.twig', [
                 'conference' => $conference,
    -            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +            'comments' => $paginator,
    +            'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,
    +            'next' => min(count($paginator), $offset + CommentRepository::PAGINATOR_PER_PAGE),
             ]));
         }
     }

O controlador obtém o ``offset`` dos parâmetros de URL da Request (``$request->query``) como um inteiro (``getInt()``),  assumindo o padrão 0 se não estiver disponível.

Os offsets ``previous`` e ``next`` são calculados com base em toda a informação que temos do paginator.

.. index::
    single: Twig;if

Finalmente, atualize o template para adicionar links para a página seguinte e para a anterior:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -6,6 +6,8 @@
         <h2>{{ conference }} Conference</h2>

         {% if comments|length > 0 %}
    +        <div>There are {{ comments|length }} comments.</div>
    +
             {% for comment in comments %}
                 {% if comment.photofilename %}
                     <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" />
    @@ -18,6 +20,13 @@

                 <p>{{ comment.text }}</p>
             {% endfor %}
    +
    +        {% if previous >= 0 %}
    +            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +        {% endif %}
    +        {% if next < comments|length %}
    +            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +        {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}

Agora você poderá navegar pelos comentários através dos links "Previous" e "Next":

.. figure:: screenshots/pagination-next.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

.. figure:: screenshots/pagination-previous.png
    :alt: /conference/1?offset=2
    :align: center
    :figclass: with-browser

Refatorando o Controlador
-------------------------

Você deve ter notado que os dois métodos em ``ConferenceController`` recebem um Environment do Twig como argumento. Em vez de injetá-lo em cada método, vamos usar uma injeção de construtor (que torna a lista de argumentos mais curta e menos redundante):

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -13,12 +13,19 @@ use Twig\Environment;

     class ConferenceController extends AbstractController
     {
    +    private $twig;
    +
    +    public function __construct(Environment $twig)
    +    {
    +        $this->twig = $twig;
    +    }
    +
         /**
          * @Route("/", name="homepage")
          */
    -    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
    +    public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($twig->render('conference/index.html.twig', [
    +        return new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    @@ -26,12 +33,12 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{id}", name="conference")
          */
    -    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    -        return new Response($twig->render('conference/show.html.twig', [
    +        return new Response($this->twig->render('conference/show.html.twig', [
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

.. sidebar:: Indo Além

    * `Documentação do Twig <https://twig.symfony.com/doc/2.x/>`_;

    * `Criando e Usando Templates <https://symfony.com/doc/current/templates.html>`_  em aplicações Symfony;

    * `Tutorial do SymfonyCasts sobre o Twig <https://symfonycasts.com/screencast/symfony/twig-recipe>`_;

    * `Funções e filtros do Twig disponíveis apenas no Symfony <https://symfony.com/doc/current/reference/twig_reference.html>`_;

    * O `controlador base AbstractController <https://symfony.com/doc/current/controller.html#the-base-controller-classes-services>`_.

.. _`Twig`: https://twig.symfony.com/
