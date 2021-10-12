Descrevendo a Estrutura de Dados
================================

.. index::
    single: Doctrine
    single: Database

Para lidar com o banco de dados a partir do PHP, vamos depender do `Doctrine`_, um conjunto de bibliotecas que ajudam os desenvolvedores a gerenciar bancos de dados:

.. code-block:: bash

    $ symfony composer req "orm:^2"

Este comando instala algumas dependências: Doctrine DBAL (uma camada de abstração de banco de dados), Doctrine ORM (uma biblioteca para manipular o conteúdo do nosso banco de dados usando objetos PHP) e Doctrine Migrations.

Configurando o Doctrine ORM
---------------------------

.. index::
    single: Doctrine;Configuration

Como o Doctrine conhece a conexão com o banco de dados? A receita do Doctrine adicionou um arquivo de configuração, ``config/packages/doctrine.yaml``, que controla seu comportamento. A principal configuração é o *DSN do banco de dados*, uma string contendo todas as informações sobre a conexão: credenciais, host, porta, etc. Por padrão, o Doctrine procura uma variável de ambiente ``DATABASE_URL``.

Entendendo as Convenções de Variável de Ambiente do Symfony
--------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Você pode definir ``DATABASE_URL`` manualmente no arquivo ``.env`` ou ``.env.local``. Na verdade, graças à receita do pacote, você verá um exemplo de ``DATABASE_URL`` em seu arquivo ``.env``. Mas como a porta local para o PostgreSQL exposta pelo Docker pode mudar, isso é bastante complicado. Há uma maneira melhor.

Em vez de codificar ``DATABASE_URL`` em um arquivo, podemos prefixar todos os comandos com ``symfony``. Isso irá detectar os serviços executados pelo Docker e/ou SymfonyCloud (quando o túnel estiver aberto) e definir a variável de ambiente automaticamente.

O Docker Compose e a SymfonyCloud funcionam perfeitamente com o Symfony graças a essas variáveis de ambiente.

.. index::
    single: Symfony CLI;var:export

Verifique todas as variáveis de ambiente expostas executando ``symfony var:export``:

.. code-block:: bash

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

Lembra-se do *nome de serviço* ``database`` usado nas configurações do Docker e da SymfonyCloud? Os nomes de serviços são usados como prefixos para definir variáveis de ambiente como ``DATABASE_URL``. Se seus serviços forem nomeados de acordo com as convenções do Symfony, nenhuma outra configuração será necessária.

.. note::

    Os bancos de dados não são os únicos serviços que se beneficiam das convenções do Symfony. O mesmo se aplica ao Mailer, por exemplo (através da variável de ambiente ``MAILER_DSN``).

Alterando o Valor Padrão de DATABASE_URL em .env
-------------------------------------------------

Ainda iremos mudar o arquivo ``.env`` para configurar o ``DATABASE_DSN`` padrão para usar o PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -26,5 +26,5 @@ APP_SECRET=7567b803de0f51b0d93e66b064cad2bf
     # 
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=5.7"
    -DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=13&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"
     ###< doctrine/doctrine-bundle ###

Por que as informações precisam ser duplicadas em dois lugares diferentes? Porque em algumas plataformas em nuvem, na *hora de fazer o build*, a URL do banco de dados pode ainda não ser conhecida, mas o Doctrine precisa saber qual o mecanismo do banco de dados para construir sua configuração. Então, o host, o nome de usuário e a senha não importam.

Criando Classes de Entidade
---------------------------

Uma conferência pode ser descrita com algumas propriedades:

* A *cidade* onde a conferência é organizada;

* O *ano* da conferência;

* Uma flag *internacional* para indicar se a conferência é local ou internacional (SymfonyLive vs. SymfonyCon).

.. index:: ! Command;make:entity

O bundle Maker pode nos ajudar a gerar uma classe (uma classe de *Entidade*) que representa uma conferência:

.. code-block:: bash
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Este comando é interativo: ele irá guiá-lo através do processo de adicionar todos os campos que você precisa. Use as seguintes respostas (a maioria delas são as padrões, assim você pode pressionar a tecla "Enter" para usá-las):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Aqui está a saída completa ao executar o comando:

.. code-block:: text
    :emphasize-lines: 8,11,14,17,22,25,28,31,36,39,42,45
    :class: ignore

     created: src/Entity/Conference.php
     created: src/Repository/ConferenceRepository.php

     Entity generated! Now let's add some fields!
     You can always add more fields later manually or by re-running this command.

     New property name (press <return> to stop adding fields):
     > city

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > year

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     > 4

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > isInternational

     Field type (enter ? to see all types) [boolean]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     >



      Success!


     Next: When you're ready, create a migration with make:migration

A classe ``Conference`` foi armazenada sob o namespace ``App\Entity\``.

O comando também gerou uma classe de *repositório* do Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\Id
    single: Annotations;@ORM\\GeneratedValue
    single: Annotations;@ORM\\Column

O código gerado se parece com o seguinte (apenas uma pequena parte do arquivo está replicada aqui):

.. code-block:: php
    :caption: src/App/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    /**
     * @ORM\Entity(repositoryClass=ConferenceRepository::class)
     */
    class Conference
    {
        /**
         * @ORM\Id()
         * @ORM\GeneratedValue()
         * @ORM\Column(type="integer")
         */
        private $id;

        /**
         * @ORM\Column(type="string", length=255)
         */
        private $city;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): self
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

Note que a classe em si é uma classe PHP simples sem sinais do Doctrine. As annotations são usadas para adicionar metadados úteis para que o Doctrine mapeie a classe para sua respectiva tabela de banco de dados.

O Doctrine adicionou uma propriedade ``id`` para armazenar a chave primária do registro na tabela de banco de dados. Essa chave (``@ORM\Id()``) é gerada automaticamente (``@ORM\GeneratedValue()``) por meio de uma estratégia que depende do mecanismo de banco de dados.

.. index::
    single: Command;make:entity

Agora, gere uma classe de Entidade para comentários da conferência:

.. code-block:: bash
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime||no)

    $ symfony console make:entity Comment

Digite as seguintes respostas:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime``, ``no``.

Relacionando Entidades
----------------------

.. index::
    single: Command;make:entity

As duas entidades, Conference e Comment, devem ser interligadas. Uma conferência pode ter zero ou mais comentários, o que é chamado de relacionamento *um-para-muitos*.

Use o comando ``make:entity`` novamente para adicionar esse relacionamento à classe ``Conference``:

.. code-block:: bash
    :class: answers(comments||OneToMany||Comment||conference||no||yes)

    $ symfony console make:entity Conference

.. code-block:: text
    :emphasize-lines: 4,7,10,15,18,27
    :class: ignore

     Your entity already exists! So let's add some new fields!

     New property name (press <return> to stop adding fields):
     > comments

     Field type (enter ? to see all types) [string]:
     > OneToMany

     What class should this entity be related to?:
     > Comment

     A new property will also be added to the Comment class...

     New field name inside Comment [conference]:
     >

     Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
     > no

     Do you want to activate orphanRemoval on your relationship?
     A Comment is "orphaned" when it is removed from its related Conference.
     e.g. $conference->removeComment($comment)

     NOTE: If a Comment may *change* from one Conference to another, answer "no".

     Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
     > yes

     updated: src/Entity/Conference.php
     updated: src/Entity/Comment.php

.. note::

    Se você digitar ``?`` como uma resposta para o tipo, você obterá todos os tipos suportados:

    .. code-block:: text
        :class: ignore

        Main types
          * string
          * text
          * boolean
          * integer (or smallint, bigint)
          * float

        Relationships / Associations
          * relation (a wizard will help you build the relation)
          * ManyToOne
          * OneToMany
          * ManyToMany
          * OneToOne

        Array/Object Types
          * array (or simple_array)
          * json
          * object
          * binary
          * blob

        Date/Time Types
          * datetime (or datetime_immutable)
          * datetimetz (or datetimetz_immutable)
          * date (or date_immutable)
          * time (or time_immutable)
          * dateinterval

        Other Types
          * decimal
          * guid
          * json_array

.. index::
    single: Annotations;@ORM\\ManyToOne
    single: Annotations;@ORM\\JoinColumn
    single: Annotations;@ORM\\OneToMany

Dê uma olhada no diff completo das classes de entidade depois de adicionar o relacionamento:

.. code-block:: diff
    :class: ignore

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -36,6 +36,12 @@ class Comment
          */
         private $createdAt;

    +    /**
    +     * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
    +     * @ORM\JoinColumn(nullable=false)
    +     */
    +    private $conference;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +94,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): self
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -31,6 +33,16 @@ class Conference
          */
         private $isInternational;

    +    /**
    +     * @ORM\OneToMany(targetEntity=Comment::class, mappedBy="conference", orphanRemoval=true)
    +     */
    +    private $comments;
    +
    +    public function __construct()
    +    {
    +        $this->comments = new ArrayCollection();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -71,4 +83,35 @@ class Conference

             return $this;
         }
    +
    +    /**
    +     * @return Collection|Comment[]
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): self
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments[] = $comment;
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): self
    +    {
    +        if ($this->comments->contains($comment)) {
    +            $this->comments->removeElement($comment);
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

Tudo o que você precisa para gerenciar o relacionamento foi gerado para você. Uma vez gerado, o código torna-se seu; sinta-se à vontade para personalizá-lo da maneira que quiser.

Adicionando mais Propriedades
-----------------------------

.. index::
    single: Command;make:entity

Acabei de perceber que nos esquecemos de adicionar uma propriedade na entidade Comment: os participantes podem querer anexar uma foto da conferência para ilustrar seus comentários.

Execute ``make:entity`` mais uma vez e adicione uma propriedade/coluna chamada ``photoFilename`` do tipo ``string``, mas permita que ela seja ``null``, já que o upload de uma foto é opcional:

.. code-block:: bash
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrando o Banco de Dados
-------------------------

.. index:: ! Command;make:migration

O modelo do projeto está agora totalmente descrito pelas duas classes geradas.

Em seguida, precisamos criar as tabelas de banco de dados relacionadas a essas entidades PHP.

*Doctrine Migrations* é a combinação perfeita para essa tarefa. Ele já foi instalado como parte da dependência ``orm``.

Uma *migração* é uma classe que descreve as alterações necessárias para atualizar um esquema de banco de dados de seu estado atual para o novo estado definido pelas annotations da entidade. Como o banco de dados está vazio por enquanto, a migração deve consistir na criação de duas tabelas.

Vamos ver o que o Doctrine gera:

.. code-block:: bash

    $ symfony console make:migration

Observe o nome do arquivo gerado na saída (um nome que se parece com ``migrations/Version20191019083640.php``):

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version20191019083640 extends AbstractMigration
    {
        public function up(Schema $schema) : void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema) : void
        {
            // ...
        }
    }

Atualizando o Banco de Dados Local
----------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Agora você pode executar a migração gerada para atualizar o esquema do banco de dados local:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

O esquema do banco de dados local agora está atualizado, pronto para armazenar alguns dados.

Atualizando o Banco de Dados de Produção
------------------------------------------

Os passos necessários para migrar o banco de dados de produção são os mesmos que você já conhece: fazer o commit das alterações e implantar.

Ao implantar o projeto, a SymfonyCloud atualiza o código, mas também executa a migração do banco de dados, se houver (ela detecta se o comando ``doctrine:migrations:migrate`` existe).

.. sidebar:: Indo Além

    * `Bases de dados e Doctrine ORM <https://symfony.com/doc/current/doctrine.html>`_ em aplicações Symfony;

    * `SymfonyCasts: tutorial sobre o Doctrine <https://symfonycasts.com/screencast/symfony-doctrine/install>`_;

    * `Trabalhando com Associações/Relações no Doctrine <https://symfony.com/doc/current/doctrine/associations.html>`_;

    * `DoctrineMigrationsBundle docs <https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html>`_.

.. _`Doctrine`: https://www.doctrine-project.org/
