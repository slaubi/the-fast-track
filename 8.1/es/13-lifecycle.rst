Gestionando el ciclo de vida de los objetos de Doctrine
=======================================================

Al crear un nuevo comentario, sería estupendo que la fecha ``createdAt`` se ajustara automáticamente con la fecha y hora actual.

Doctrine tiene diferentes maneras de manipular los objetos y sus propiedades durante su ciclo de vida (antes de que se cree la fila en la base de datos, después de que se actualice la fila...)

Definiendo *callbacks* del ciclo de vida
----------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

Cuando el comportamiento no necesita ningún servicio y debe ser aplicado a un solo tipo de entidad, define un *callback* en la clase de la entidad:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -57,8 +57,6 @@ class CommentCrudController extends AbstractCrudController
             ]);
             if (Crud::PAGE_EDIT === $pageName) {
                 yield $createdAt->setFormTypeOption('disabled', true);
    -        } else {
    -            yield $createdAt;
             }
         }
     }
    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -86,6 +87,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue(): void
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

El *evento* ``ORM\PrePersist`` se lanza cuando el objeto se almacena en la base de datos por primera vez. Cuando esto sucede, se llama al método ``setCreatedAtValue()`` y se utiliza la fecha y hora actual para el valor de la propiedad ``createdAt``.

Agregando *slugs* a las conferencias
------------------------------------

Las URLs de las conferencias no son útiles: ``/conference/1``. Y lo que es más importante, dependen de un detalle de implementación (queda expuesta la clave primaria de la base de datos).

¿Qué tal si en su lugar usamos URLs como ``/conference/paris-2020``? Eso se vería mucho mejor. ``paris-2020`` es lo que llamamos el *slug* de la conferencia.

.. index::
    single: Command;make:entity

Añade una nueva propiedad ``slug`` para las conferencias (una cadena de 255 caracteres que no permita valores nulos):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Crea un archivo de migración para agregar la nueva columna:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

Y ejecuta esa nueva migración:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

¿Te has encontrado con un error? Era de esperar. ¿Por qué? Porque pedimos que el *slug* no aceptara valores ``null`` pero las entradas existentes en la base de datos de la conferencia tendrán un valor ``null`` cuando se ejecute la migración. Arreglemos eso ajustando la migración:

.. code-block:: diff
    :caption: patch_file

    --- i/migrations/Version00000000000000.php
    +++ w/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema): void

El truco aquí es agregar la columna y permitirle que acepte valores ``null``, luego asignar a *slug* un valor no ``null``, y finalmente, cambiar la columna de *slug* para no permitir valores ``null``.

.. note::

    Para un proyecto real, el uso de ``CONCAT(LOWER(city), '-', year)`` puede que no sea suficiente. En ese caso, necesitaríamos usar el Slugger "verdadero".

.. index::
    single: Command;doctrine:migrations:migrate

La migración debería funcionar bien ahora:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

Debido a que la aplicación pronto usará *slugs* para encontrar cada conferencia, ajustemos la entidad Conference para asegurar que los valores de *slug* sean únicos en la base de datos:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -6,8 +6,10 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    +#[UniqueEntity('slug')]
     class Conference
     {
         #[ORM\Id]
    @@ -30,7 +32,7 @@ class Conference
         #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

Como habrás adivinado, necesitamos realizar la danza de la migración:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Generando *slugs*
-----------------

.. index::
    single: Components;String
    single: Slug

Generar un *slug* que se lea bien en una URL (donde cualquier cosa que no sean caracteres ASCII debe ser codificada) es una tarea desafiante, especialmente para idiomas que no sean el inglés. Por ejemplo, ¿Cómo conviertes ``é`` a ``e``?

En lugar de reinventar la rueda, usemos el componente de Symfony ``String``, que facilita la manipulación de las cadenas y proporciona un *slugger*.

Añade un método ``computeSlug()`` a la clase ``Conference`` que calcule el *slug* basado en los datos de la conferencia:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -50,6 +51,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger): void
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

El método ``computeSlug()`` sólo calcula un *slug* cuando el actual está vacío o ajustado al valor especial ``-``. ¿Por qué necesitamos el valor especial ``-``? Porque cuando se agrega una conferencia en el *backend*, se requiere el *slug*. Por lo tanto, necesitamos un valor no vacío que le diga a la aplicación que queremos que el *slug* se genere automáticamente.

Definiendo un *callback* de ciclo de vida complejo
--------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

Al igual que con la propiedad ``createdAt``, el ``slug`` debe ser configurado automáticamente cada vez que se actualice la conferencia llamando al método ``computeSlug()``.

Pero como este método depende de una implementación ``SluggerInterface``, no podemos añadir un evento ``prePersist`` como antes (no tenemos una forma de inyectar el slugger).

En su lugar, crea un oyente de entidades de Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\PrePersistEventArgs;
    use Doctrine\ORM\Event\PreUpdateEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
        }

        public function prePersist(Conference $conference, PrePersistEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, PreUpdateEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }
    }

Ten en cuenta que el *slug* se actualiza cuando se crea una nueva conferencia (``prePersist()``) y cuando se actualiza (``preUpdate()``).

Configurando un servicio en el contenedor
-----------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Hasta ahora, no hemos hablado de un componente clave de Symfony, el *contenedor de inyección de dependencias*. El contenedor se encarga de gestionar *los servicios*: crearlos e inyectarlos cuando sea necesario.

Un *servicio* es un objeto "global" que proporciona características (por ejemplo, un *mailer*, un *logger*, un *slugger*, etc.) a diferencia de los *objetos de datos* (por ejemplo, instancias de entidades de Doctrine).

Rara vez interactúas con el contenedor directamente, ya que inyecta automáticamente objetos de esos servicios siempre que los necesites: cuando indicas el nombre de la clase que provee el servicio (*type-hinting*), el contenedor inyecta los objetos en los parámetros del controlador.

Si te preguntabas cómo se registró el oyente del evento en el paso anterior, ahora tienes la respuesta: el contenedor. Cuando una clase implementa algunas interfaces específicas, el contenedor sabe que la clase necesita ser registrada de cierta manera.

Aquí, como nuestra clase no implementa ninguna interfaz ni extiende ninguna clase base, Symfony no sabe cómo autoconfigurarla. En su lugar, podemos usar un atributo para indicarle al contenedor de Symfony cómo cablearla:

.. code-block:: diff
    :caption: patch_file

    --- i/src/EntityListener/ConferenceEntityListener.php
    +++ w/src/EntityListener/ConferenceEntityListener.php
    @@ -3,10 +3,14 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\PrePersistEventArgs;
     use Doctrine\ORM\Event\PreUpdateEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    No confundas a los oyentes de los eventos de Doctrine con los de Symfony. Aunque parezcan muy similares, no están utilizando la misma infraestructura realmente.

Usando *slugs* en la aplicación
--------------------------------

Intenta añadir más conferencias en el módulo de servicio y cambia la ciudad o el año de una existente; el *slug* no se actualizará excepto si utilizas el valor especial ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

El último cambio es actualizar los controladores y las plantillas para utilizar el ``slug`` de la conferencia en lugar del ``id`` de la conferencia para las rutas. Como el parámetro de la ruta ya no es la clave primaria de la entidad, indícale a ``#[MapEntity]`` qué propiedad debe coincidir pasando un ``mapping`` explícito:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -20,7 +20,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    #[Route('/conference/{slug}', name: 'conference')]
    +    public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -16,7 +16,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
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

El acceso a las páginas de la conferencia debe realizarse ahora a través de su *slug*:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Yendo más allá

    * El `sistema de eventos de Doctrine`_ (callbacks del ciclo de vida y oyentes, oyentes de entidades y suscriptores del ciclo de vida);

    * La documentación del `componente String`_ ;

    * El `contenedor de servicio`_ ;

    * La `Chuleta de servicios de Symfony`_ .

.. _`sistema de eventos de Doctrine`: https://symfony.com/doc/current/doctrine/events.html
.. _`componente String`: https://symfony.com/doc/current/components/string.html
.. _`contenedor de servicio`: https://symfony.com/doc/current/service_container.html
.. _`Chuleta de servicios de Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
