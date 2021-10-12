Gerenciando o Ciclo de Vida de Objetos Doctrine
===============================================

Seria ótimo atribuir automaticamente data e hora atual na propriedade ``createdAt`` ao criar um novo comentário.

O Doctrine tem diferentes maneiras de manipular objetos e suas propriedades durante seu ciclo de vida (antes que o registro no banco de dados seja criado, após um registro ser atualizado, ...).

Definindo Callbacks do Ciclo de Vida
------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

Quando o comportamento não precisa de nenhum serviço e se aplica a apenas um tipo de entidade, defina um callback na classe da entidade:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\ORM\Mapping as ORM;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    + * @ORM\HasLifecycleCallbacks()
      */
     class Comment
     {
    @@ -106,6 +107,14 @@ class Comment
             return $this;
         }

    +    /**
    +     * @ORM\PrePersist
    +     */
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTime();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

O *evento* ``@ORM\PrePersist`` é disparado quando o objeto é persistido no banco de dados pela primeira vez. Quando isso ocorre, o método ``setCreatedAtValue()`` é chamado, atribuindo data e hora atuais como valor da propriedade ``createdAt``.

Adicionando Slugs às Conferências
-----------------------------------

As URLs das conferências não são amigáveis: ``/conference/1``. Mais importante: elas dependem de um detalhe da implementação (a chave primária do banco de dados está exposta).

Ao invés disso, que tal usar URLs como ``/conference/paris-2020``? Assim parece bem melhor. ``paris-2020`` é o que chamamos de *slug* da conferência.

.. index::
    single: Command;make:entity

Adicione uma nova propriedade ``slug`` para conferências (uma string de valor obrigatório, not null, de 255 caracteres):

.. code-block:: bash
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Crie um arquivo de migração para adicionar a nova coluna:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

E execute essa nova migração:

.. code-block:: bash
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Obteve um erro? Isto é esperado. Por quê? Porque definimos o slug como ``not null``, mas no momento em que a migração é executada, as entradas já existentes no banco de dados da conferência têm ``null`` como valor. Vamos corrigir isso ajustando a migração:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714152808 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema) : void

O truque aqui é adicionar a coluna e permitir que ela seja ``null``, então definir o slug para um valor diferente de ``null`` e, finalmente, alterar a coluna slug para não permitir ``null``.

.. note::

    Para um projeto real, usar ``CONCAT(LOWER(city), '-', year)`` pode não ser suficiente. Nesse caso, precisaríamos usar o "verdadeiro" Slugger.

.. index::
    single: Command;doctrine:migrations:migrate

A migração deve executar corretamente agora:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

Como a aplicação irá em breve usar slugs para encontrar cada conferência, vamos ajustar a entidade Conference para garantir que os valores atribuídos ao slug sejam únicos no banco de dados:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,9 +6,11 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    + * @UniqueEntity("slug")
      */
     class Conference
     {
    @@ -40,7 +42,7 @@ class Conference
         private $comments;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, unique=true)
          */
         private $slug;

.. index::
    single: Command;make:migration

Como você pode imaginar, precisaremos fazer a dança da migração:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Gerando Slugs
-------------

.. index::
    single: Components;String
    single: Slug

Gerar um slug legível em uma URL (onde qualquer coisa além de caracteres ASCII deve ser codificada) é uma tarefa desafiadora, especialmente para idiomas que não o inglês. Como você converte ``é`` para ``e``, por exemplo?

Em vez de reinventar a roda, vamos usar o componente Symfony ``String``, que facilita a manipulação de strings e fornece um *slugger*:

.. code-block:: bash

    $ symfony composer req string

Adicione um método ``computeSlug()`` à classe ``Conference`` que gera o slug com base nos dados da conferência:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    @@ -61,6 +62,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger)
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

O método ``computeSlug()`` só tenta gerar um slug quando o atual está vazio ou definido para o valor especial ``-``. Por que precisamos do valor especial ``-``? Porque ao adicionar uma conferência no backend, o slug é obrigatório. Então, precisamos de um valor não vazio que informe à aplicação que queremos que o slug seja gerado automaticamente.

Definindo um Callback Complexo do Ciclo de Vida
-----------------------------------------------

.. index::
    single: Doctrine;Entity Listener

Semelhante ao que vimos com a propriedade ``createdAt``, a propriedade ``slug`` deverá ser definida automaticamente sempre que a conferência for atualizada, chamando o método ``computeSlug()``.

Mas como esse método depende de uma implementação de ``SluggerInterface``, não podemos adicionar o evento ``prePersist``, conforme feito anteriormente (pois não temos uma maneira de injetar o slugger).

Em vez disso, crie um listener de entidade do Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        private $slugger;

        public function __construct(SluggerInterface $slugger)
        {
            $this->slugger = $slugger;
        }

        public function prePersist(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }
    }

Note que o slug é atualizado quando uma nova conferência é criada (``prePersist()``) e sempre que ela é atualizada (``preUpdate()``).

Configurando um Serviço no Container
-------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Até agora, não falamos sobre um componente fundamental do Symfony, o *container de injeção de dependência*. O container é responsável por gerenciar os *serviços*: criá-los e injetá-los sempre que necessário.

Um *serviço* é um objeto "global" que provê funcionalidades (por exemplo: um mailer, um logger, um slugger, etc.) diferente dos *objetos de dados* (como instâncias de entidades do Doctrine, por exemplo).

Você raramente interage diretamente com o container, pois ele injeta automaticamente objetos de serviço sempre que você precisa deles: o container injeta os objetos como argumentos no controlador quando você os define com declaração de tipo, por exemplo.

Se você se perguntou como o listener de eventos foi registrado no passo anterior, agora você tem a resposta: o container. Quando uma classe implementa algumas interfaces específicas, o container sabe que a classe precisa ser registrada de uma certa maneira.

Infelizmente, a automação não é fornecida para tudo, especialmente para pacotes de terceiros. O listener de entidade que acabamos de escrever é um desses exemplos; ele não pode ser gerenciado automaticamente pelo container de serviços do Symfony, pois ele não implementa nenhuma interface e não estende uma "classe conhecida".

Precisamos declarar parcialmente o listener no container. A ligação de dependência pode ser omitida, já que ainda pode ser adivinhada pelo container, mas precisamos adicionar manualmente algumas *tags* para registrar o listener com o dispatcher de eventos do Doctrine:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -29,3 +29,7 @@ services:

         # add more service definitions when explicit configuration is needed
         # please note that last definitions always *replace* previous ones
    +    App\EntityListener\ConferenceEntityListener:
    +        tags:
    +            - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
    +            - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}

.. note::

    Não confunda listeners de eventos do Doctrine com os do Symfony. Mesmo que sejam muito parecidos, eles não utilizam a mesma infraestrutura por baixo dos panos.

Usando Slugs na Aplicação
---------------------------

Tente adicionar mais conferências no backend e altere a cidade ou o ano de uma já existente; o slug não será atualizado, a não ser que você use o valor especial ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;@Route

A última alteração é atualizar os controladores e os templates para utilizar o ``slug`` da conferência para as rotas em vez do ``id``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -31,7 +31,7 @@ class ConferenceController extends AbstractController
         }

         /**
    -     * @Route("/conference/{id}", name="conference")
    +     * @Route("/conference/{slug}", name="conference")
          */
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -10,7 +10,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -22,10 +22,10 @@
             {% endfor %}

             {% if previous >= 0 %}
    -            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
             {% endif %}
             {% if next < comments|length %}
    -            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
             {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>

O acesso às páginas da conferência agora deve ser feito através do seu slug:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Indo Além

    * O `sistema de eventos do Doctrine <https://symfony.com/doc/current/doctrine/events.html>`_ (listeners e callbacks do ciclo de vida, listeners de entidade e subscribers do ciclo de vida);

    * The `String component docs <https://symfony.com/doc/current/components/string.html>`_;

    * O `Container de Serviço <https://symfony.com/doc/current/service_container.html>`_;

    * A `Cheat Sheet dos Serviços do Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_.
