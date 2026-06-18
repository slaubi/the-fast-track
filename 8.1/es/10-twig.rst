Construyendo la interfaz de usuario
===================================

.. index::
    single: Twig
    single: Templates

Ya está todo listo para crear la primera versión de la interfaz de usuario del sitio web. Por el momento no lo haremos bonito, solo funcional.

¿Recuerdas la llamada a ``htmlspecialchars()`` que tuvimos que hacer en el controlador para el huevo de pascua con el fin de evitar problemas de seguridad?  Esa es la razón por la que no usaremos PHP para nuestras plantillas. En su lugar, usaremos Twig. Además de ocuparse por nosotros de cambiar las secuencias de caracteres peligrosas por su equivalente inofensivo (*escaping*), `Twig`_ trae un montón de características interesantes que vamos a aprovechar, como la herencia de plantillas.

Usando Twig para las plantillas
-------------------------------

.. index::
    single: Twig;Layout
    single: Twig;block

Todas las páginas del sitio web compartirán el mismo diseño y distribución de elementos principales (*layout*). Al instalar Twig, se ha creado un directorio ``templates/`` automáticamente y también se ha incluido un *layout* de ejemplo en ``base.html.twig``.

.. code-block:: html+twig
    :caption: templates/base.html.twig
    :class: ignore

    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
            {% block stylesheets %}
            {% endblock %}

            {% block javascripts %}
                {% block importmap %}{{ importmap('app') }}{% endblock %}
            {% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
        </body>
    </html>

Un *layout* puede definir elementos ``block``, que son los lugares donde las *plantillas hijo* que *extienden* del layout añaden sus contenidos.

.. index::
    single: Twig;extends
    single: Twig;for

Vamos a crear una plantilla para la página principal del proyecto en ``templates/conference/index.html.twig``:

.. code-block:: html+twig
    :caption: templates/conference/index.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook{% endblock %}

    {% block body %}
        <h2>Give your feedback!</h2>

        {% for conference in conferences %}
            <h4>{{ conference }}</h4>
        {% endfor %}
    {% endblock %}

La plantilla *extiende* ``base.html.twig`` y redefine los bloques ``body`` y ``title``.

.. index::
    single: Twig;Syntax

La notación ``{% %}`` en una plantilla indica *acciones* y *estructura*.

La notación ``{{ }}`` se utiliza para *mostrar* algo. ``{{ conference }}`` muestra la representación de la conferencia (el resultado de llamar a ``__toString`` en el objeto ``Conference``).

Usando Twig en un controlador
-----------------------------

Actualiza el controlador para renderizar la plantilla Twig:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,22 +2,19 @@

     namespace App\Controller;

    +use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    +use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response(<<<EOF
    -            <html>
    -                <body>
    -                    <img src="/images/under-construction.gif" />
    -                </body>
    -            </html>
    -            EOF
    -        );
    +        return new Response($twig->render('conference/index.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
         }
     }

Aquí están pasando un montón de cosas.

Para poder renderizar una plantilla, necesitamos el objeto ``Environment`` de Twig (el principal punto de entrada de Twig). Observa que solicitamos la instancia de Twig especificándola como parámetro en el método del controlador. Symfony es lo suficientemente inteligente para saber cómo inyectar el objeto correcto.

También necesitamos el repositorio de la entidad *Conference* para obtener todas las conferencias que hay en la base de datos.

En el código del controlador, el método ``render()`` construye y pasa un *array* de variables a la plantilla. Estamos pasando la lista de objetos ``Conference`` como una variable ``conferences``.

Un controlador es una clase estándar de PHP. Ni siquiera necesitamos extender la clase ``AbstractController`` si queremos ser explícitos sobre nuestras dependencias. Puedes omitir esa extensión (pero no lo hagas, ya que usaremos los útiles atajos que proporciona en los próximos pasos).

Creando la página para una conferencia
---------------------------------------

Cada conferencia debe tener una página donde se listen sus comentarios. Añadir una nueva página es cuestión de agregar un controlador, asignarle una ruta y crear la plantilla relacionada.

Agrega un método ``show()`` en ``src/Controller/ConferenceController.php`` :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,6 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Conference;
    +use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    @@ -17,4 +20,13 @@ final class ConferenceController extends AbstractController
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    +
    +    #[Route('/conference/{id}', name: 'conference')]
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    {
    +        return new Response($twig->render('conference/show.html.twig', [
    +            'conference' => $conference,
    +            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +        ]));
    +    }
     }

Este método tiene un comportamiento especial que aún no hemos visto. Pedimos que se inyecte una instancia de ``Conference`` en el método. Pero puede haber muchos de estos en la base de datos. El atributo ``#[MapEntity]`` le indica a Symfony que obtenga el correcto basándose en el ``{id}`` enviado en la ruta de la solicitud (siendo ``id`` la llave primaria de la tabla ``conference`` en la base de datos).

Los comentarios relacionados con la conferencia se pueden obtener a través del método ``findBy()`` que toma un criterio de búsqueda como primer argumento.

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;for
    single: Twig;if
    single: Twig;else
    single: Twig;asset
    single: Twig;format_datetime
    single: Twig;length

El último paso es crear el fichero: ``templates/conference/show.html.twig``

.. code-block:: html+twig
    :caption: templates/conference/show.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

    {% block body %}
        <h2>{{ conference }} Conference</h2>

        {% if comments|length > 0 %}
            {% for comment in comments %}
                {% if comment.photofilename %}
                    <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
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

En esta plantilla, estamos usando la notación ``|`` para llamar a los *filtros* de Twig. Un filtro transforma un valor. ``comments|length`` devuelve el número de comentarios y ``comment.createdAt|format_datetime('medium', 'short')`` da formato a la fecha en una representación legible para el ser humano.

Intenta obtener la "primera" conferencia a través de ``/conference/1``, y observa el siguiente error:

.. figure:: screenshots/intl-twig-error.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

El error viene del filtro ``format_datetime`` ya que no es parte del núcleo de Twig. El mensaje de error te da una pista sobre qué paquete debe instalarse para solucionar el problema:

.. code-block:: terminal

    $ symfony composer req "twig/intl-extra:^3"

Ahora la página funciona correctamente.

Vinculando las páginas
-----------------------

.. index::
    single: Twig;Link
    single: Link

El último paso para terminar nuestra primera versión de la interfaz de usuario es vincular las páginas de la conferencia desde la página principal:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -7,5 +7,8 @@

         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
    +        <p>
    +            <a href="/conference/{{ conference.id }}">View</a>
    +        </p>
         {% endfor %}
     {% endblock %}

Sin embargo, especificar en el código una ruta de forma literal es una mala idea por varios motivos. El más importante de ellos es que si cambiara la ruta en el futuro (por ejemplo de ``/conference/{id}`` a ``/conferences/{id}``), todos los vínculos deberían actualizarse manualmente.

.. index::
    single: Twig;path

En su lugar, utiliza la *función* ``path()`` de Twig y usa el *nombre de la ruta*:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="/conference/{{ conference.id }}">View</a>
    +            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}

La función ``path()`` genera la ruta a una página utilizando su nombre de ruta. Los valores de los parámetros de la misma se pasan como un mapa de Twig.

Paginando los comentarios
-------------------------

.. index::
    single: Doctrine;Paginator
    single: Paginator

Con miles de asistentes, podemos esperar bastantes comentarios. Si los mostramos todos en una sola página, crecerá muy rápido.

Crea un método llamado ``getCommentPaginator()`` en la clase  ``CommentRepository`` que devuelva un objeto *Paginator* de comentarios basado en una conferencia y un *offset* (valor de inicio por el que empezar a listar):

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -3,19 +3,37 @@
     namespace App\Repository;

     use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
      * @extends ServiceEntityRepository<Comment>
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    public const COMMENTS_PER_PAGE = 2;
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
    +            ->setMaxResults(self::COMMENTS_PER_PAGE)
    +            ->setFirstResult($offset)
    +            ->getQuery()
    +        ;
    +
    +        return new Paginator($query);
    +    }
    +
         //    /**
         //     * @return Comment[] Returns an array of Comment objects
         //     */

Hemos establecido el número máximo de comentarios por página en 2 para facilitar las pruebas.

Para gestionar la paginación en la plantilla, pasa a Twig el objeto tipo Paginator de Doctrine en lugar del tipo Collection:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,6 +8,7 @@ use App\Repository\ConferenceRepository;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
     use Twig\Environment;

    @@ -22,11 +23,15 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
    +        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
    +
             return new Response($twig->render('conference/show.html.twig', [
                 'conference' => $conference,
    -            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +            'comments' => $paginator,
    +            'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
    +            'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
             ]));
         }
     }

El atributo ``#[MapQueryParameter]`` mapea el parámetro ``offset`` de la cadena de consulta al argumento ``$offset`` del controlador, usando ``0`` por defecto cuando no está definido. Como el offset proviene del cliente, la opción ``min_range`` valida que no sea negativo; Symfony devuelve una respuesta 404 cuando el valor no es válido.

Los intervalos de ``previous`` (anterior) y ``next`` (siguiente) se calculan en base a toda la información que tenemos del paginador.

.. index::
    single: Twig;if

Por último, actualiza la plantilla para añadir enlaces a las páginas siguiente y anterior:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -6,6 +6,8 @@
         <h2>{{ conference }} Conference</h2>

         {% if comments|length > 0 %}
    +        <div>There are {{ comments|length }} comments.</div>
    +
             {% for comment in comments %}
                 {% if comment.photofilename %}
                     <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
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

Ahora podrás navegar por los comentarios a través de los enlaces "Previous" y "Next":

.. figure:: screenshots/pagination-next.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

.. figure:: screenshots/pagination-previous.png
    :alt: /conference/1?offset=2
    :align: center
    :figclass: with-browser

Refactorizando el controlador
-----------------------------

Habrás notado que ambos métodos en ``ConferenceController`` toman un entorno (*environment*) de Twig como argumento. En lugar de inyectarlo en cada método, aprovechemos el método auxiliar ``render()`` proporcionado por la clase padre:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,28 +9,27 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
    -use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
    +    public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($twig->render('conference/index.html.twig', [
    +        return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]));
    +        ]);
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    -        return new Response($twig->render('conference/show.html.twig', [
    +        return $this->render('conference/show.html.twig', [
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    -        ]));
    +        ]);
         }
     }

.. sidebar:: Yendo más allá

    * `Documentación de Twig`_ ;

    * `Creando y usando plantillas`_ en aplicaciones Symfony;

    * `Tutorial de Twig en SymfonyCasts`_ ;

    * `Funciones y filtros Twig disponibles solo en Symfony`_ ;

    * El `controlador base AbstractController`_ .

.. _`Twig`: https://twig.symfony.com/
.. _`Documentación de Twig`: https://twig.symfony.com/doc/3.x/
.. _`Creando y usando plantillas`: https://symfony.com/doc/current/templates.html
.. _`Tutorial de Twig en SymfonyCasts`: https://symfonycasts.com/screencast/symfony/twig-recipe
.. _`Funciones y filtros Twig disponibles solo en Symfony`: https://symfony.com/doc/current/reference/twig_reference.html
.. _`controlador base AbstractController`: https://symfony.com/doc/current/controller.html#the-base-controller-classes-services
