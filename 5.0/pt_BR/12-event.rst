Escutando Eventos
=================

Falta ao layout atual um cabeçalho de navegação para voltar à página inicial ou alternar de uma conferência para a seguinte.

Adicionando um Cabeçalho ao Site
---------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Tudo o que deve ser exibido em todas as páginas web, como um cabeçalho, deve fazer parte do layout base principal:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -6,6 +6,15 @@
             {% block stylesheets %}{% endblock %}
         </head>
         <body>
    +        <header>
    +            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    +            <ul>
    +            {% for conference in conferences %}
    +                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +            {% endfor %}
    +            </ul>
    +            <hr />
    +        </header>
             {% block body %}{% endblock %}
             {% block javascripts %}{% endblock %}
         </body>

Adicionar este código ao layout significa que todos os templates que o estendem devem definir uma variável ``conferences``, e esta deve ser criada e passada a partir dos respectivos controladores.

As we only have two controllers, you might do the following:

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -32,9 +32,10 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             return new Response($this->twig->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
             ]));

Imagine ter que atualizar dezenas de controladores. E ter que fazer o mesmo em todos os novos. Isso não é muito prático. Deve haver uma maneira melhor.

Twig tem o conceito de variáveis globais. Uma *variável global* está disponível em todos os templates renderizados. Você pode defini-las em um arquivo de configuração, mas ele só funciona com valores estáticos. Para adicionar todas as conferências como uma variável global do Twig, vamos criar um listener.

Descobrindo os Eventos do Symfony
---------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

O Symfony vem com um Componente Event Dispatcher. Um dispatcher *despacha* certos *eventos* em momentos específicos aos quais os *listeners* podem escutar. Os listeners são ganchos para o núcleo do framework.

Por exemplo, alguns eventos permitem que você interaja com o ciclo de vida das requisições HTTP. Durante o tratamento de uma requisição, o dispatcher despacha eventos quando uma requisição é criada, quando um controlador está prestes a ser executado, quando uma resposta está pronta para ser enviada ou quando uma exceção foi lançada. Um *listener* pode escutar um ou mais eventos e executar alguma lógica baseada no contexto do evento.

Os eventos são pontos de extensão bem definidos que tornam o framework mais genérico e extensível. Muitos Componentes do Symfony, como Security, Messenger, Workflow ou Mailer os utilizam extensivamente.

Outro exemplo pronto de eventos e listeners em ação é o ciclo de vida de um comando: você pode criar um listener para executar código antes da execução de *qualquer* comando.

Qualquer pacote ou bundle também pode despachar seus próprios eventos para tornar seu código extensível.

Para evitar um arquivo de configuração que descreva quais eventos um listener pode escutar, crie um *subscriber*. Um subscriber é um listener com um método estático ``getSubscribedEvents()`` que retorna sua configuração. Isso permite que os subscribers sejam registrados automaticamente no dispatcher do Symfony.

Implementando um Subscriber
---------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Agora que você já sabe a música de cor, use o bundle Maker para gerar um subscriber:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

O comando pergunta qual o evento que você deseja escutar. Escolha o evento ``Symfony\Component\HttpKernel\Event\ControllerEvent``, que é disparado logo antes do controlador ser chamado. É o melhor momento para injetar a variável global ``conferences`` para que o Twig tenha acesso a ela quando o controlador renderizar o template. Atualize o seu subscriber da seguinte forma:

.. code-block:: diff
    :caption: patch_file

    --- a/src/EventSubscriber/TwigEventSubscriber.php
    +++ b/src/EventSubscriber/TwigEventSubscriber.php
    @@ -2,14 +2,25 @@

     namespace App\EventSubscriber;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\EventSubscriberInterface;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     class TwigEventSubscriber implements EventSubscriberInterface
     {
    +    private $twig;
    +    private $conferenceRepository;
    +
    +    public function __construct(Environment $twig, ConferenceRepository $conferenceRepository)
    +    {
    +        $this->twig = $twig;
    +        $this->conferenceRepository = $conferenceRepository;
    +    }
    +
         public function onControllerEvent(ControllerEvent $event)
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }

         public static function getSubscribedEvents()

Agora você pode adicionar quantos controladores quiser: a variável ``conferences`` estará sempre disponível no Twig.

.. note::

    Falaremos sobre uma alternativa muito melhor em termos de desempenho em uma etapa posterior.

Ordenando as Conferências por Ano e Cidade
-------------------------------------------

Ordenar a lista de conferências por ano pode facilitar a navegação. Poderíamos criar um método personalizado para recuperar e ordenar todas as conferências, mas em vez disso, vamos sobrescrever a implementação padrão do método ``findAll()`` para garantir que a ordenação se aplique a todas as chamadas:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/ConferenceRepository.php
    +++ b/src/Repository/ConferenceRepository.php
    @@ -19,6 +19,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll()
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         // /**
         //  * @return Conference[] Returns an array of Conference objects
         //  */

Ao final desta etapa, o site deve ter a seguinte aparência:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Indo Além

    * O `Fluxo de Requisição e Resposta <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ em aplicações Symfony;

    * Os `eventos HTTP nativos do Symfony <https://symfony.com/doc/current/reference/events.html>`_;

    * Os `eventos nativos do Console do Symfony <https://symfony.com/doc/current/components/console/events.html>`_.
