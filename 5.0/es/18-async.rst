Volviéndonos asíncronos
=========================

.. index::
    single: Async

La comprobación del correo no deseado durante la gestión del envío del formulario puede dar lugar a algunos problemas. Si la API de Akismet se vuelve lenta, nuestro sitio web también lo será para los usuarios. Pero lo que es peor, si tenemos un timeout o si la API de Akismet no está disponible, podemos perder comentarios.

Lo ideal sería que almacenáramos los datos enviados sin publicarlos y devolviéramos inmediatamente una respuesta. La comprobación de spam se puede hacer de forma separada.

Marcando comentarios
--------------------

.. index::
    single: Command;make:entity

Tenemos que introducir un ``state`` para los comentarios: ``submitted``, ``spam`` y ``published``.

Agrega la propiedad ``state`` a la clase ``Comment``:

.. code-block:: bash
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Command;make:migration

Crea una migración de base de datos:

.. code-block:: bash

    $ symfony console make:migration

Modifica la migración para actualizar todos los comentarios existentes para que sean ``published`` de forma predeterminada:

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

Migra la base de datos:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\Column

También debemos asegurarnos de que, por defecto, el valor de ``state`` sea ``submitted``:

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

Actualiza la configuración de EasyAdmin para poder ver el estado del comentario:

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

No olvides también actualizar las pruebas configurando el ``state`` de los fixtures:

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

Para las pruebas del controlador, simula la validación:

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

A partir de una prueba de PHPUnit, puedes obtener cualquier servicio del contenedor a través de ``self::$container->get()``; también da acceso a servicios no públicos.

Entendiendo Messenger
---------------------

.. index::
    single: Messenger
    single: Components;Messenger

El componente Messenger es el encargado de la gestión de código asíncrono cuando usamos Symfony:

.. code-block:: bash

    $ symfony composer req messenger

Cuando alguna lógica deba ser ejecutada de forma asíncrona, se envía un mensaje (*message*) a un bus de mensajería (*message bus*). El bus almacena el mensaje en una cola (*queue*) y vuelve inmediatamente para permitir que el flujo de operaciones se reanude lo más rápido posible.

Un *consumidor* se ejecuta continuamente en segundo plano para leer nuevos mensajes en la cola y ejecutar la lógica asociada. El consumidor puede ejecutarse en el mismo servidor que la aplicación web o en un servidor separado.

Es muy similar a la forma en que se manejan las peticiones HTTP, excepto que no tenemos respuestas.

Creando un manejador de mensajes (*Message handler*)
----------------------------------------------------

Un mensaje es una clase de objeto de datos que no debe contener ninguna lógica. Será serializado para ser almacenado en una cola, por lo que sólo se almacenarán datos serializables "simples".

Crea la clase ``CommentMessage``:

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

En el mundo de Messenger, no tenemos controladores, sino manejadores de mensajes.

Crea una clase ``CommentMessageHandler`` bajo un nuevo espacio de nombres ``App\MessageHandler`` que sepa cómo manejar los mensajes ``CommentMessage``:

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

``MessageHandlerInterface`` es una interfaz que actúa como *marcador*. Sólo ayuda a Symfony a auto-registrarse y a auto-configurar la clase como un manejador de Messenger. Por convención, la lógica de un manejador vive en un método llamado ``__invoke()``. El *type-hint* ``CommentMessage`` en el argumento de este método le dice a Messenger qué clase manejará.

Actualiza el controlador para utilizar el nuevo sistema:

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

En lugar de depender del Spam Checker, ahora enviamos un mensaje al bus. El manejador entonces decide qué hacer con él.

Hemos logrado algo inesperado. Hemos desacoplado nuestro controlador del Spam Checker y hemos movido la lógica a una nueva clase, el manejador. Es un caso de uso perfecto para el bus. Prueba el código, funciona. Todo se sigue haciendo sincrónicamente, pero el código probablemente ya sea "mejor".

Restringiendo comentarios visualizados
--------------------------------------

Actualiza la lógica de visualización para evitar que aparezcan comentarios no publicados en el frontend:

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

Volviéndonos asíncronos de verdad
-----------------------------------

.. index::
    single: RabbitMQ

Por defecto, los manejadores son llamados sincrónicamente. Para ser asíncronos, es necesario configurar explícitamente qué cola utilizar para cada manejador en el fichero de configuración ``config/packages/messenger.yaml``:

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

La configuración le dice al bus que envíe instancias de ``App\Message\CommentMessage`` a la cola ``async``, que está definida por un DSN, almacenado en la variable de entorno ``RABBITMQ_DSN``.

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

.. sidebar:: Volcado y restauración de la base de datos

    Never call ``docker-compose down`` if you don't want to lose data. Or backup first. Use ``pg_dump`` to dump the database data:

    .. code-block:: bash
        :class: ignore

        $ symfony run pg_dump --data-only > dump.sql

    And restore the data:

    .. code-block:: bash
        :class: ignore

        $ symfony run psql < dump.sql

Consumiendo mensajes
--------------------

Si intentas enviar un nuevo comentario, ya no se llamará al verificador de spam. Agrega una llamada ``error_log()`` en el método ``getSpamScore()`` para confirmar. En su lugar, un mensaje está esperando en RabbitMQ, listo para ser consumido por algunos procesos.

.. index::
    single: Command;messenger:consume

Como te puedes imaginar, Symfony viene con un comando ``consumer`` para el consumidor. Ejecútalo ahora:

.. code-block:: bash
    :class: ignore

    $ symfony console messenger:consume async -vv

Debe consumir inmediatamente el mensaje despachado para el comentario enviado:

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

La actividad del consumidor de mensajes se registra, pero obtendrás información en tiempo real en la consola si le pasas el parámetro ``-vv``. Incluso deberías ser capaz de detectar la llamada a la API de Akismet.

Para detener al consumidor, pulsa ``Ctrl+C``.

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

Ejecutando *workers* en segundo plano
-------------------------------------

En lugar de lanzar al consumidor cada vez que publicamos un comentario y detenerlo inmediatamente después, queremos ejecutarlo continuamente sin tener demasiadas ventanas de terminal o pestañas abiertas.

El comando Symfony puede administrar dichos comandos en segundo plano o *workers* (trabajadores) usando el parámetro de *daemon* (``-d``) en el comando ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Ejecuta de nuevo el consumidor del mensaje, pero envíalo en segundo plano:

.. code-block:: bash

    $ symfony run -d --watch=config,src,templates,vendor symfony console messenger:consume async

La opción ``--watch`` le dice a Symfony que el comando debe ser reiniciado siempre que haya un cambio en el sistema de archivos en los directorios ``config/``, ``src/``, ``templates/`` o ``vendor/``.

.. note::

    No uses ``-vv`` ya que obtendrías mensajes duplicados en ``server:log`` (los mensajes registrados y los mensajes de consola).

Si el consumidor dejara de trabajar por alguna razón (por quedarse sin memoria, por un fallo...), se reiniciará automáticamente. Y si el consumidor falla demasiado rápido, el comando Symfony desistirá de reiniciarlo.

.. index::
    single: Symfony CLI;server:log

Usando ``symfony server:log`` los registros generados se unirán a todos los demás registros procedentes de PHP, el servidor web y la aplicación:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Utiliza el comando ``server:status`` para listar todos los *workers* en segundo plano pertenecientes al proyecto actual:

.. code-block:: bash
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Para detener a un *worker*, detén el servidor web o mata el proceso que tiene el PID que se muestra con el comando ``server:status``:

.. code-block:: bash
    :class: ignore

    $ kill 15774

Reintentando cuando los mensajes fallan
---------------------------------------

¿Qué pasa si Akismet se cae mientras se consume un mensaje? Las personas que envían comentarios no notarán nada, pero el mensaje se pierde y el spam no se controla.

Messenger tiene un mecanismo de reintento para cuando ocurre una excepción mientras se maneja un mensaje. Vamos a configurarlo:

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

Si ocurre un problema mientras se maneja un mensaje, el consumidor volverá a intentarlo 3 veces antes de darse por vencido. Pero en lugar de descartar el mensaje, lo almacenará en un almacenamiento más permanente, la cola ``failed``, que utiliza la base de datos Doctrine.

Inspecciona los mensajes fallidos y vuelve a intentarlo mediante los siguientes comandos:

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

Ejecutando *workers* en SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;Workers
    single: Workers

Para consumir mensajes de RabbitMQ, necesitamos ejecutar el comando ``messenger:consume`` continuamente. En SymfonyCloud, este es el rol de un *worker*:

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

Al igual que hace el comando Symfony, SymfonyCloud también gestiona los reinicios y los registros.

.. index::
    single: Symfony CLI;logs

Para obtener *logs* de un *worker*, usa:

.. code-block:: bash
    :class: ignore

    $ symfony logs --worker=messages all

.. sidebar:: Yendo más allá

    * `Tutorial de SymfonyCasts Messenger <https://symfonycasts.com/screencast/messenger>`_;

    * La arquitectura del `Enterprise service bus <https://en.wikipedia.org/wiki/Enterprise_service_bus>`_ y el `patrón CQRS <https://martinfowler.com/bliki/CQRS.html>`_;

    * La `documentación de Symfony Messenger <https://symfony.com/doc/current/messenger.html>`_;

    * `RabbitMQ docs <https://www.rabbitmq.com/documentation.html>`_.
