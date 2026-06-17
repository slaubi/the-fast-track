Exponiendo una API con API Platform
===================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Hemos terminado la implementación del sitio web del Libro de Visitas. Para dar más juego con los datos que tenemos, ¿qué te parece si los exponemos mediante una API? Una aplicación móvil podría utilizar esta API para mostrar todas las conferencias, los comentarios y posiblemente permitir a los asistentes enviar comentarios.

En este paso, vamos a implementar una API de sólo lectura.

Instalando API Platform
-----------------------

Es posible exponer una API escribiendo algo de código, pero si queremos usar estándares, es mejor utilizar una solución que se encargue del trabajo pesado. Una solución como API Platform:

.. code-block:: bash

    $ symfony composer req api

Exponiendo una API para las conferencias
----------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;Groups

Solo necesitamos unas pocas anotaciones en la clase Conference para configurar la API:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,16 +2,25 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiResource;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
      * @UniqueEntity("slug")
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="conference:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="conference:item"}}},
    + *     order={"year"="DESC", "city"="ASC"},
    + *     paginationEnabled=false
    + * )
      */
     class Conference
     {
    @@ -20,21 +29,25 @@ class Conference
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $city;

         /**
          * @ORM\Column(type="string", length=4)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $year;

         /**
          * @ORM\Column(type="boolean")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $isInternational;

         /**
    @@ -45,6 +58,7 @@ class Conference
         /**
          * @ORM\Column(type="string", length=255, unique=true)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $slug;

         public function __construct()

La anotación principal ``@ApiResource`` configura la API de conferencias. Restringe las operaciones posibles a ``get`` y configura varias cosas: como qué campos mostrar y cómo ordenar las conferencias.

Por defecto, el punto de entrada principal para la API es ``/api`` debido a la configuración que añadió en ``config/routes/api_platform.yaml`` la receta durante la instalación del paquete.

Una interfaz web te permite interactuar con la API:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Utilízala para probar la funcionalidad que ofrece:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

¡Imagina el tiempo que te llevaría implementar todo esto desde cero!

Exponiendo una API para los comentarios
---------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;@ApiFilter
    single: Annotations;Groups

Haz lo mismo con los comentarios:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,13 +2,26 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiFilter;
    +use ApiPlatform\Core\Annotation\ApiResource;
    +use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
      * @ORM\HasLifecycleCallbacks()
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="comment:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="comment:item"}}},
    + *     order={"createdAt"="DESC"},
    + *     paginationEnabled=false
    + * )
    + *
    + * @ApiFilter(SearchFilter::class, properties={"conference": "exact"})
      */
     class Comment
     {
    @@ -17,18 +30,21 @@ class Comment
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $author;

         /**
          * @ORM\Column(type="text")
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $text;

         /**
    @@ -36,22 +52,26 @@ class Comment
          */
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $email;

         /**
          * @ORM\Column(type="datetime")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $createdAt;

         /**
          * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
          * @ORM\JoinColumn(nullable=false)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $conference;

         /**
          * @ORM\Column(type="string", length=255, nullable=true)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $photoFilename;

         /**

Utilizaremos el mismo tipo de anotaciones para configurar la clase.

Restringiendo los comentarios expuestos por la API
--------------------------------------------------

De forma predeterminada, API Platform expone todos los registros de la base de datos. Pero para los comentarios, solo aquellos que estén publicados deberían ser parte de la API.

Cuando necesites restringir los ítems devueltos por la API, crea un servicio que implemente ``QueryCollectionExtensionInterface`` para controlar la consulta de Doctrine utilizada para las colecciones y/o ``QueryItemExtensionInterface`` para controlar los ítems:

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 13-15,20-22

    namespace App\Api;

    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }
    }

La clase de extensión de consultas aplica su lógica sólo para el recurso ``Comment`` y modifica el *Doctrine query builder* para considerar únicamente los comentarios que están en estado ``published`` (publicado).

Configurando CORS
-----------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Por defecto, la política de seguridad *same-origin* (mismo-origen) de los clientes HTTP modernos prohíbe llamar a la API desde otro dominio. El paquete CORS, instalado como parte de ``composer req api``, envía encabezados de CORS (Cross-Origin Resource Sharing) basados en la variable de entorno ``CORS_ALLOW_ORIGIN``.

Por defecto, su valor, definido en ``.env``, permite realizar peticiones HTTP desde ``localhost`` y ``127.0.0.1`` en cualquier puerto. Eso es exactamente lo que necesitamos para el siguiente paso, en el que crearemos una SPA (*Single Page Application*) que tendrá su propio servidor web y que consultará a la API.

.. sidebar:: Yendo más allá

    * `Tutorial de API Platform de SymfonyCasts <https://symfonycasts.com/screencast/api-platform>`_;

    * Para habilitar el soporte de GraphQL, ejecuta ``composer require webonyx/graphql-php``, y luego abre ``/api/graphql`` en el navegador.
