Gestionando el ciclo de vida de los objetos de Doctrine
=======================================================

Al crear un nuevo comentario, sería estupendo que la fecha ``createdAt`` se ajustara automáticamente con la fecha y hora actual.

Doctrine tiene diferentes maneras de manipular los objetos y sus propiedades durante su ciclo de vida (antes de que se cree la fila en la base de datos, después de que se actualice la fila...)

Definiendo *callbacks* del ciclo de vida
----------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

Cuando el comportamiento no necesita ningún servicio y debe ser aplicado a un solo tipo de entidad, define un *callback* en la clase de la entidad:

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

El *evento* ``@ORM\PrePersist`` se lanza cuando el objeto se almacena en la base de datos por primera vez. Cuando esto sucede, se llama al método ``setCreatedAtValue()`` y se utiliza la fecha y hora actual para el valor de la propiedad ``createdAt``.

Agregando *slugs* a las conferencias
------------------------------------

Las URLs de las conferencias no son útiles: ``/conference/1``. Y lo que es más importante, dependen de un detalle de implementación (queda expuesta la clave primaria de la base de datos).

¿Qué tal si en su lugar usamos URLs como ``/conference/paris-2020``? Eso se vería mucho mejor. ``paris-2020`` es lo que llamamos el *slug* de la conferencia.

.. index::
    single: Command;make:entity

Añade una nueva propiedad ``slug`` para las conferencias (una cadena de 255 caracteres que no permita valores nulos):

.. code-block:: bash
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Crea un archivo de migración para agregar la nueva columna:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

Y ejecuta esa nueva migración:

.. code-block:: bash
    :class: ignore

    $ symfony console doctrine:migrations:migrate

¿Te has encontrado con un error? Era de esperar. ¿Por qué? Porque pedimos que el *slug* no aceptara valores ``nulos`` pero las entradas existentes en la base de datos de la conferencia tendrán un valor ``nulo`` cuando se ejecute la migración. Arreglemos eso ajustando la migración:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
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

El truco aquí es agregar la columna y permitirle que acepte valores ``nulos``, luego asignar a *slug* un valor no ``nulo``, y finalmente, cambiar la columna de *slug* para no permitir valores ``nulos``.

.. note::

    Para un proyecto real, el uso de ``CONCAT(LOWER(city), '-', year)`` puede que no sea suficiente. En ese caso, necesitaríamos usar el Slugger "verdadero".

.. index::
    single: Command;doctrine:migrations:migrate

La migración debería funcionar bien ahora:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

Debido a que la aplicación pronto usará *slugs* para encontrar cada conferencia, ajustemos la entidad Conference para asegurar que los valores de *slug* sean únicos en la base de datos:

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
    single: Components;Validator

Debido a que utilizamos un validador para garantizar la unicidad de los slugs, necesitamos agregar el componente Symfony Validator:

.. code-block:: bash

    $ symfony composer req validator

.. index::
    single: Command;make:migration

Como habrás adivinado, necesitamos realizar la danza de la migración:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Generando *slugs*
-----------------

.. index::
    single: Components;String
    single: Slug

Generar un *slug* que se lea bien en una URL (donde cualquier cosa que no sean caracteres ASCII debe ser codificada) es una tarea desafiante, especialmente para idiomas que no sean el inglés. Por ejemplo, ¿Cómo conviertes ``é`` a ``e``?

En lugar de reinventar la rueda, usemos el componente de Symfony ``String``, que facilita la manipulación de las cadenas y proporciona un *slugger*:

.. code-block:: bash

    $ symfony composer req string

Añade un método ``computeSlug()`` a la clase ``Conference`` que calcule el *slug* basado en los datos de la conferencia:

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

Desafortunadamente, la automatización no está prevista para todo, especialmente para los paquetes de terceros. El oyente de entidades que acabamos de escribir es un ejemplo de ello; no puede ser gestionado automáticamente por el contenedor de servicios de Symfony ya que no implementa ninguna interfaz y no extiende una "clase bien conocida".

Necesitamos declarar parcialmente al oyente en el contenedor. El cableado de dependencias se puede omitir ya que todavía se puede adivinar por el contenedor, pero necesitamos agregar manualmente algunas *etiquetas* para registrar al oyente con el despachador de eventos de Doctrine:

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

    No confundas a los oyentes de los eventos de Doctrine con los de Symfony. Aunque parezcan muy similares, no están utilizando la misma infraestructura realmente.

Usando *slugs* en la aplicación
--------------------------------

Intenta añadir más conferencias en el módulo de servicio y cambia la ciudad o el año de una existente; el *slug* no se actualizará excepto si utilizas el valor especial ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;Route

El último cambio es actualizar los controladores y las plantillas para utilizar el ``slug`` de la conferencia en lugar del ``id`` de la conferencia para las rutas:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
             ]));
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    +    #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -18,7 +18,7 @@
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

El acceso a las páginas de la conferencia debe realizarse ahora a través de su *slug*:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Yendo más allá

    * El `sistema de eventos de Doctrine <https://symfony.com/doc/current/doctrine/events.html>`_ (callbacks del ciclo de vida y oyentes, oyentes de entidades y suscriptores del ciclo de vida);

    * La documentación del `componente String  <https://symfony.com/doc/current/components/string.html>`_ ;

    * El `contenedor de servicio <https://symfony.com/doc/current/service_container.html>`_ ;

    * La `Chuleta de servicios de Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_.
