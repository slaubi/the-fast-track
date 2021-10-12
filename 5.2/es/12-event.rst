Escuchando eventos
==================

Al diseño actual le falta un encabezado de navegación para poder volver a la página principal o cambiar de una conferencia a la siguiente.

Añadiendo un encabezado al sitio web
------------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Cualquier cosa que deba mostrarse en todas las páginas web, como un encabezado, debe formar parte del diseño (*layout*) base principal:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -14,6 +14,15 @@
             {% endblock %}
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
         </body>
     </html>

Añadir este código al *layout* hace que todas las plantillas que lo extienden tengan que definir una variable ``conferences``, que se debe crear y pasar desde sus controladores.

Como solo tenemos dos controladores, *podrías* hacer lo siguiente (no apliques el cambio a tu código ya que aprenderemos una forma mejor muy pronto):

.. code-block:: diff
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -29,12 +29,13 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return new Response($this->twig->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

Imagina tener que actualizar docenas de controladores. Y hacer lo mismo con todos los nuevos. Esto no es muy práctico. Debe haber una manera mejor.

Twig tiene la noción de variables globales. Una *variable global* está disponible en todas las plantillas renderizadas. Puedes definirlas en un archivo de configuración, pero sólo funciona para valores estáticos. Para añadir todas las conferencias como una variable global de Twig, vamos a crear un *listener*.

Descubriendo los eventos de Symfony
-----------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony viene de serie con un componente para lanzar eventos. Un despachador (*dispatcher*) *dispara* ciertos *eventos* en momentos específicos que los *oyentes* (*listeners*) pueden escuchar. Los oyentes son ganchos que nos conectan con los procesos internos del *framework*.

Por ejemplo, algunos eventos te permiten interactuar con el ciclo de vida de las peticiones HTTP. Durante la gestión de una solicitud, el despachador envía eventos cuando se ha creado una solicitud, cuando un controlador está a punto de ser ejecutado, cuando una respuesta está lista para ser enviada o cuando se ha lanzado una excepción. Un *oyente* puede escuchar uno o más eventos y ejecutar alguna lógica basada en el contexto del evento.

Los eventos son puntos de extensión bien definidos que hacen que el *framework* sea más genérico y extensible. Muchos componentes de Symfony como Security, Messenger, Workflow o Mailer los utilizan continuamente.

Otro ejemplo de eventos y oyentes en acción es el ciclo de vida de un comando: puedes crear un oyente para que ejecute código antes de que se ejecute *cualquier* comando.

Cualquier paquete o *bundle* también puede enviar sus propios eventos para hacer que su código sea extensible.

Para evitar tener un archivo de configuración que describa qué eventos quiere escuchar un oyente, crea un *suscriptor* (*subscriber*). Un suscriptor es un oyente con un método estático ``getSubscribedEvents()`` que devuelve su configuración. Esto permite que los suscriptores se registren en el despachador de Symfony automáticamente.

Implementando un suscriptor
---------------------------

.. index::
    single: Event;Subscriber
    single: Subscriber
    single: Event;Listener
    single: Listener
    single: Command;make:subscriber

Ahora te sabes la canción de memoria, usa el *bundle* maker para generar un suscriptor:

.. code-block:: bash
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:subscriber TwigEventSubscriber

El comando te pregunta sobre el evento que deseas escuchar. Selecciona el evento ``Symfony\Component\HttpKernel\Event\ControllerEvent``, que se envía justo antes de que se llame al controlador. Es el mejor momento para inyectar la variable ``conferences`` global para que Twig tenga acceso a ella cuando el controlador llame a la plantilla. Actualiza tu suscriptor de la siguiente manera:

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

Ahora, puedes añadir tantos controladores como quieras: la variable ``conferences`` siempre estará disponible en Twig.

.. note::

    Hablaremos de una alternativa mucho mejor en cuanto a rendimiento en un paso posterior.

Ordenando conferencias por año y ciudad
---------------------------------------

Ordenar la lista de conferencias por año puede facilitar la navegación. Podríamos crear un método personalizado para recuperar y clasificar todas las conferencias, pero en su lugar, vamos a anular la implementación predeterminada del método ``findAll()`` para asegurarnos de que la clasificación se aplica en todas partes:

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

Tras acabar este paso, el sitio web debe tener el siguiente aspecto:

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Yendo más allá

    * El `flujo de Request-Response <https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request>`_ en las aplicaciones Symfony;

    * Los `eventos HTTP incorporados en Symfony <https://symfony.com/doc/current/reference/events.html>`_ ;

    * Los `eventos Console incorporados en Symfony <https://symfony.com/doc/current/components/console/events.html>`_ .
