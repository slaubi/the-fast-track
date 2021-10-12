Tornando Assíncrono
====================

.. index::
    single: Async

Verificar a existência de spam durante o processamento da submissão do formulário pode levar a alguns problemas. Se a API do Akismet ficar lenta, nosso site também ficará lento para os usuários. Pior ainda, se atingirmos um tempo limite ou se a API do Akismet estiver indisponível, poderemos perder comentários.

Idealmente, devemos armazenar os dados enviados sem publicá-los e retornar imediatamente uma resposta. A verificação de spam pode então ser feita em outro momento.

Adicionando Flags aos Comentários
----------------------------------

.. index::
    single: Command;make:entity

Precisamos introduzir um ``state`` para os comentários: ``submitted``, ``spam`` e ``published``.

Adicione uma propriedade ``state`` na classe ``Comment``:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Crie uma migração do banco de dados:

.. code-block:: bash

    $ symfony console make:migration

Modifique a migração para atualizar todos os comentários existentes para serem ``published`` por padrão:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714155905 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE comment ADD state VARCHAR(255)');
    +        $this->addSql("UPDATE comment SET state='published'");
    +        $this->addSql('ALTER TABLE comment ALTER COLUMN state SET NOT NULL');
         }

         public function down(Schema $schema) : void

.. index::
    single: Command;doctrine:migrations:migrate

Migre o banco de dados:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

Também devemos garantir que, por padrão, o ``state`` seja definido como ``submitted``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -55,9 +55,9 @@ class Comment
         private $photoFilename;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, options={"default": "submitted"})
          */
    -    private $state;
    +    private $state = 'submitted';

         public function __toString(): string
         {

Atualize a configuração do EasyAdmin para poder ver o state do comentário:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/easy_admin.yaml
    +++ b/config/packages/easy_admin.yaml
    @@ -18,6 +18,7 @@ easy_admin:
                         - author
                         - { property: 'email', type: 'email' }
                         - { property: 'photoFilename', type: 'image', 'base_path': "/uploads/photos", label: 'Photo' }
    +                    - state
                         - { property: 'createdAt', type: 'datetime' }
                     sort: ['createdAt', 'ASC']
                     filters: ['conference']
    @@ -26,5 +27,6 @@ easy_admin:
                         - { property: 'conference' }
                         - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                         - 'author'
    +                    - { property: 'state' }
                         - { property: 'email', type: 'email' }
                         - text

Não se esqueça de atualizar também os testes, definindo ``state`` nas fixtures:

.. code-block:: diff
    :caption: patch_file

    --- a/src/DataFixtures/AppFixtures.php
    +++ b/src/DataFixtures/AppFixtures.php
    @@ -37,8 +37,16 @@ class AppFixtures extends Fixture
             $comment1->setAuthor('Fabien');
             $comment1->setEmail('fabien@example.com');
             $comment1->setText('This was a great conference.');
    +        $comment1->setState('published');
             $manager->persist($comment1);

    +        $comment2 = new Comment();
    +        $comment2->setConference($amsterdam);
    +        $comment2->setAuthor('Lucas');
    +        $comment2->setEmail('lucas@example.com');
    +        $comment2->setText('I think this one is going to be moderated.');
    +        $manager->persist($comment2);
    +
             $admin = new Admin();
             $admin->setRoles(['ROLE_ADMIN']);
             $admin->setUsername('admin');

.. index::
    single: Test;Container
    single: Container;Test

Para os testes do controlador, simule a validação:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -2,6 +2,8 @@

     namespace App\Tests\Controller;

    +use App\Repository\CommentRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

     class ConferenceControllerTest extends WebTestCase
    @@ -22,10 +24,16 @@ class ConferenceControllerTest extends WebTestCase
             $client->submitForm('Submit', [
                 'comment_form[author]' => 'Fabien',
                 'comment_form[text]' => 'Some feedback from an automated functional test',
    -            'comment_form[email]' => 'me@automat.ed',
    +            'comment_form[email]' => $email = 'me@automat.ed',
                 'comment_form[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
             ]);
             $this->assertResponseRedirects();
    +
    +        // simulate comment validation
    +        $comment = self::$container->get(CommentRepository::class)->findOneByEmail($email);
    +        $comment->setState('published');
    +        self::$container->get(EntityManagerInterface::class)->flush();
    +
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }

A partir de um teste do PHPUnit, você pode obter qualquer serviço do container via ``self::$container->get()``; ele também dá acesso a serviços não públicos.

Entendendo o Messenger
----------------------

.. index::
    single: Messenger
    single: Components;Messenger

Gerenciar código assíncrono com o Symfony é tarefa do Componente Messenger:

.. code-block:: bash

    $ symfony composer req messenger

Quando alguma lógica deve ser executada de forma assíncrona, envie uma *mensagem* para um *barramento de mensagens*. O barramento armazena a mensagem em uma *fila* e retorna imediatamente para permitir que o fluxo de operações seja retomado o mais rápido possível.

Um *consumidor* é executado continuamente em segundo plano para ler novas mensagens na fila e executar a lógica associada. O consumidor pode executar no mesmo servidor que a aplicação web ou em um servidor separado.

É muito semelhante à forma como as requisições HTTP são tratadas, exceto que não temos respostas.

Programando um Manipulador de Mensagens
---------------------------------------

Uma mensagem é uma classe de objeto de dados que não deve conter nenhuma lógica. Ela será serializada para ser armazenada em uma fila, portanto, armazene somente dados serializáveis "simples".

Crie a classe ``CommentMessage``:

.. code-block:: php
    :caption: src/Message/CommentMessage.php

    namespace App\Message;

    class CommentMessage
    {
        private $id;
        private $context;

        public function __construct(int $id, array $context = [])
        {
            $this->id = $id;
            $this->context = $context;
        }

        public function getId(): int
        {
            return $this->id;
        }

        public function getContext(): array
        {
            return $this->context;
        }
    }

No mundo do Messenger, não temos controladores, mas manipuladores de mensagens.

Crie uma classe ``CommentMessageHandler`` sob um novo namespace ``App\MessageHandler`` que saiba como manipular mensagens ``CommentMessage``:

.. code-block:: php
    :caption: src/MessageHandler/CommentMessageHandler.php

    namespace App\MessageHandler;

    use App\Message\CommentMessage;
    use App\Repository\CommentRepository;
    use App\SpamChecker;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Component\Messenger\Handler\MessageHandlerInterface;

    class CommentMessageHandler implements MessageHandlerInterface
    {
        private $spamChecker;
        private $entityManager;
        private $commentRepository;

        public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository)
        {
            $this->entityManager = $entityManager;
            $this->spamChecker = $spamChecker;
            $this->commentRepository = $commentRepository;
        }

        public function __invoke(CommentMessage $message)
        {
            $comment = $this->commentRepository->find($message->getId());
            if (!$comment) {
                return;
            }

            if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
                $comment->setState('spam');
            } else {
                $comment->setState('published');
            }

            $this->entityManager->flush();
        }
    }

``MessageHandlerInterface`` é uma interface *marcadora*. Ela só ajuda o Symfony a auto-registrar e auto-configurar a classe como um manipulador de mensagens. Por convenção, a lógica de um manipulador reside em um método chamado ``__invoke()``. A declaração de tipo ``CommentMessage`` no único argumento desse método diz ao Messenger qual classe ele irá manipular.

Atualize o controlador para usar o novo sistema:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -5,14 +5,15 @@ namespace App\Controller;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use App\Form\CommentFormType;
    +use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;

    @@ -20,11 +21,13 @@ class ConferenceController extends AbstractController
     {
         private $twig;
         private $entityManager;
    +    private $bus;

    -    public function __construct(Environment $twig, EntityManagerInterface $entityManager)
    +    public function __construct(Environment $twig, EntityManagerInterface $entityManager, MessageBusInterface $bus)
         {
             $this->twig = $twig;
             $this->entityManager = $entityManager;
    +        $this->bus = $bus;
         }

         /**
    @@ -40,7 +43,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -58,6 +61,7 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -65,11 +69,8 @@ class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    -                throw new \RuntimeException('Blatant spam, go away!');
    -            }

    -            $this->entityManager->flush();
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

Em vez de depender do Verificador de Spam, despachamos agora uma mensagem ao barramento. O manipulador decide então o que fazer com ela.

Conseguimos algo inesperado. Desacoplamos nosso controlador do Verificador de Spam e movemos a lógica para uma nova classe, o manipulador. É um caso de uso perfeito para o barramento. Teste o código, ele funciona. Tudo ainda é feito de forma síncrona, mas o código provavelmente já está "melhor".

Restringindo os Comentários Exibidos
-------------------------------------

Atualize a lógica de exibição para evitar que os comentários não publicados apareçam no frontend:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -27,7 +27,9 @@ class CommentRepository extends ServiceEntityRepository
         {
             $query = $this->createQueryBuilder('c')
                 ->andWhere('c.conference = :conference')
    +            ->andWhere('c.state = :state')
                 ->setParameter('conference', $conference)
    +            ->setParameter('state', 'published')
                 ->orderBy('c.createdAt', 'DESC')
                 ->setMaxResults(self::PAGINATOR_PER_PAGE)
                 ->setFirstResult($offset)

Tornando Realmente Assíncrono
------------------------------

.. index::
    single: RabbitMQ

Por padrão, os manipuladores são chamados de forma síncrona. Para tornar assíncrono, você precisa configurar explicitamente qual fila usar para cada manipulador no arquivo de configuração ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,10 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            # async: '%env(MESSENGER_TRANSPORT_DSN)%'
    +            async: '%env(RABBITMQ_DSN)%'
                 # failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

             routing:
                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

A configuração diz ao barramento para enviar instâncias de ``App\Message\CommentMessage`` para a fila ``async``, que é definida por um DSN, armazenado na variável de ambiente ``RABBITMQ_DSN``.

Adding RabbitMQ to the Docker Stack
-----------------------------------

.. index::
    single: Docker;RabbitMQ

As you might have guessed, we are going to use RabbitMQ:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -12,3 +12,7 @@ services:
         redis:
             image: redis:5-alpine
             ports: [6379]
    +
    +    rabbitmq:
    +        image: rabbitmq:3.7-management
    +        ports: [5672, 15672]

Restarting Docker Services
--------------------------

To force Docker Compose to take the RabbitMQ container into account, stop the containers and restart them:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

.. code-block:: bash
    :class: hide

    $ sleep 10

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

.. sidebar:: Fazendo Dump e Restaurando os Dados do Banco de Dados

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

Consumindo Mensagens
--------------------

Se você tentar enviar um novo comentário, o verificador de spam não será mais chamado. Adicione uma chamada ``error_log()`` no método ``getSpamScore()`` para confirmar. Em vez disso, uma mensagem está esperando no RabbitMQ, pronta para ser consumida por alguns processos.

.. index::
    single: Command;messenger:consume

Como você pode imaginar, o Symfony vem com um comando consumidor. Execute-o agora:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Ele deve consumir imediatamente a mensagem despachada quando o comentário foi submetido:

.. code-block:: text
    :class: ignore

     [OK] Consuming messages from transports "async".

     // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

     // Quit the worker with CONTROL-C.

    11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
    11:30:20 INFO      [http_client] Request: "POST https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [http_client] Response: "200 https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
    11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]

A atividade do consumidor da mensagem é registrada no log, mas você recebe feedback instantâneo no console, passando a flag ``-vv``. Você deve ser capaz até de identificar a chamada à API do Akismet.

Para parar o consumidor, pressione ``Ctrl+C``.

Exploring the RabbitMQ Web Management Interface
-----------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

If you want to see queues and messages flowing through RabbitMQ, open its web management interface:

.. code-block:: bash
    :class: ignore

    $ symfony open:local:rabbitmq

Or from the web debug toolbar:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Use ``guest``/``guest`` to login to the RabbitMQ management UI:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

Executando Workers em Segundo Plano
-----------------------------------

Em vez de iniciar o consumidor toda vez que publicarmos um comentário e o pararmos logo depois, queremos executá-lo continuamente sem ter muitas janelas ou abas do terminal abertas.

A CLI do Symfony pode gerenciar esses comandos em segundo plano ou workers usando a flag daemon (``-d``) no comando ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Execute o consumidor de mensagem novamente, mas envie-o em segundo plano:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

A opção ``--watch`` diz ao Symfony que o comando deve ser reiniciado sempre que houver uma alteração no sistema de arquivos nos diretórios ``config/``, ``src/``, ``templates/`` ou ``vendor/``.

.. note::

    Não use ``-vv``, pois você duplicaria as mensagens em ``server:log`` (mensagens registradas no log e mensagens do console).

Se o consumidor parar de funcionar por alguma razão (limite de memória, bug, ...), ele será reiniciado automaticamente. E se o consumidor falhar muito rápido, a CLI do Symfony irá desistir.

.. index::
    single: Symfony CLI;server:log

Os logs são transmitidos através do ``symfony server:log`` junto com todos os outros logs provenientes do PHP, do servidor web e da aplicação:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Use o comando ``server:status`` para listar todos os workers em segundo plano gerenciados para o projeto atual:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Para parar um worker, pare o servidor web ou mate o processo com o PID fornecido pelo comando ``server:status``:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Repetindo Mensagens com Falha
-----------------------------

E se o Akismet estiver inacessível enquanto consumimos uma mensagem? Não há impacto para as pessoas que enviam comentários, mas a mensagem é perdida e o spam não é verificado.

O Messenger tem um mecanismo para tentar novamente quando uma exceção ocorre durante a manipulação de uma mensagem. Vamos configurá-lo:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/messenger.yaml
    +++ b/config/packages/messenger.yaml
    @@ -5,10 +5,17 @@ framework:

             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
    -            async: '%env(RABBITMQ_DSN)%'
    -            # failed: 'doctrine://default?queue_name=failed'
    +            async:
    +                dsn: '%env(RABBITMQ_DSN)%'
    +                retry_strategy:
    +                    max_retries: 3
    +                    multiplier: 2
    +
    +            failed: 'doctrine://default?queue_name=failed'
                 # sync: 'sync://'

    +        failure_transport: failed
    +
             routing:
                 # Route your messages to the transports
                 App\Message\CommentMessage: async

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

Se ocorrer um problema durante a manipulação de uma mensagem, o consumidor tentará novamente 3 vezes antes de desistir. Mas, ao invés de descartar a mensagem, ele irá armazená-la em um armazenamento mais permanente, a fila ``failed``, que usa o banco de dados Doctrine.

Inspecione as mensagens com falha e tente novamente através dos seguintes comandos:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Deploying RabbitMQ
------------------

.. index::
    single: SymfonyCloud;RabbitMQ
    single: RabbitMQ

Adding RabbitMQ to the production servers can be done by adding it to the list of services:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -5,3 +5,8 @@ db:

     rediscache:
         type: redis:5.0
    +
    +queue:
    +    type: rabbitmq:3.7
    +    disk: 1024
    +    size: S

Reference it in the web container configuration as well and enable the ``amqp`` PHP extension:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - amqp
             - redis
             - pdo_pgsql
             - apcu
    @@ -26,6 +27,7 @@ disk: 512
     relationships:
         database: "db:postgresql"
         redis: "rediscache:redis"
    +    rabbitmq: "queue:rabbitmq"

     web:
         locations:

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

When the RabbitMQ service is installed on a project, you can access its web management interface by opening the tunnel first:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony tunnel:close

Executando Workers na SymfonyCloud
----------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Para consumir mensagens do RabbitMQ, precisamos executar o comando ``messenger:consume`` continuamente. Na SymfonyCloud, esse é o papel de um *worker*:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -54,3 +54,8 @@ hooks:
             set -x -e

             (>&2 symfony-deploy)
    +
    +workers:
    +    messages:
    +        commands:
    +            start: symfony console messenger:consume async -vv --time-limit=3600 --memory-limit=128M

Assim como na CLI do Symfony, a SymfonyCloud gerencia reinicializações e logs.

.. index::
    single: Symfony CLI;logs

Para obter os logs de um worker, use:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Indo Além

    * `Tutorial do SymfonyCasts sobre o Messenger <https://symfonycasts.com/screencast/messenger>`_;

    * A arquitetura `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ e o `padrão CQRS <https://martinfowler.com/bliki/CQRS.html>`_;

    * A `documentação do Messenger do Symfony <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
