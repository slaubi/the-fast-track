Een API beschikbaar maken met API platform
==========================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

We zijn klaar met de implementatie van de website voor het gastenboek. Om meer gebruik van de gegevens mogelijk te maken, zouden we een API beschikbaar kunnen stellen. Een API zou door een mobiele applicatie gebruikt kunnen worden om alle conferenties en hun reacties weer te geven en de bezoekers misschien de optie te geven om een reactie achter te laten.

In deze stap gaan we een alleen-lezen-API implementeren.

API Platform installeren
------------------------

We kunnen een API beschikbaar stellen door wat code te schrijven, maar als we gebruik willen maken van standaarden, kunnen we beter een oplossing gebruiken die al het zware werk verricht. Een oplossing zoals API Platform:

.. code-block:: terminal

    $ symfony composer req api

Een API voor conferenties beschikbaar stellen
---------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Een paar attributen op de Conference-class is alles wat we nodig hebben om de API te configureren:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,35 +2,48 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiResource;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    +#[ApiResource(
    +    collectionOperations: ['get' => ['normalization_context' => ['groups' => 'conference:list']]],
    +    itemOperations: ['get' => ['normalization_context' => ['groups' => 'conference:item']]],
    +    order: ['year' => 'DESC', 'city' => 'ASC'],
    +    paginationEnabled: false,
    +)]
     class Conference
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column(type: 'integer')]
    +    #[Groups(['conference:list', 'conference:item'])]
         private $id;

         #[ORM\Column(type: 'string', length: 255)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private $city;

         #[ORM\Column(type: 'string', length: 4)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private $year;

         #[ORM\Column(type: 'boolean')]
    +    #[Groups(['conference:list', 'conference:item'])]
         private $isInternational;

         #[ORM\OneToMany(mappedBy: 'conference', targetEntity: Comment::class, orphanRemoval: true)]
         private $comments;

         #[ORM\Column(type: 'string', length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private $slug;

         public function __construct()

De belangrijkste attribuut, ``ApiResource``, configureert de API voor conferenties. Het beperkt de mogelijke handelingen tot ``get`` en configureert verschillende dingen, zoals welke velden moeten worden weergegeven en in welke volgorde de conferenties moeten staan.

Standaard is ``/api`` het belangrijkste toegangspunt voor de API dankzij de configuratie ``config/routes/api_platform.yaml`` die door de recipe van de package is toegevoegd.

Een webinterface stelt je in staat om te communiceren met de API:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Gebruik het om de verschillende mogelijkheden te testen:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Stel je eens voor hoe lang het zou duren om dit alles vanaf nul uit te bouwen!

Een API voor reacties beschikbaar stellen
-----------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

Doe hetzelfde voor reacties:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,40 +2,58 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiFilter;
    +use ApiPlatform\Core\Annotation\ApiResource;
    +use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    +#[ApiResource(
    +    collectionOperations: ['get' => ['normalization_context' => ['groups' => 'comment:list']]],
    +    itemOperations: ['get' => ['normalization_context' => ['groups' => 'comment:item']]],
    +    order: ['createdAt' => 'DESC'],
    +    paginationEnabled: false,
    +)]
    +#[ApiFilter(SearchFilter::class, properties: ['conference' => 'exact'])]
     class Comment
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column(type: 'integer')]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $id;

         #[ORM\Column(type: 'string', length: 255)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $author;

         #[ORM\Column(type: 'text')]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $text;

         #[ORM\Column(type: 'string', length: 255)]
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $email;

         #[ORM\Column(type: 'datetime_immutable')]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $createdAt;

         #[ORM\ManyToOne(targetEntity: Conference::class, inversedBy: 'comments')]
         #[ORM\JoinColumn(nullable: false)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $conference;

         #[ORM\Column(type: 'string', length: 255, nullable: true)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $photoFilename;

         #[ORM\Column(type: 'string', length: 255, options: ["default" => "submitted"])]

Dezelfde soort attributen worden gebruikt om de class te configureren.

API restricties opleggen bij de reacties
----------------------------------------

Standaard geeft API Platform alle gegevens uit de database vrij. Maar eigenlijk moeten alleen de gepubliceerde reacties deel van de API zijn.

Om te beperken welke items door de API worden teruggestuurd, maak je een service aan die de ``QueryCollectionExtensionInterface`` implementeert, om de Doctrine-query te beheren die gebruikt wordt voor collections, en/of de ``QueryItemExtensionInterface``, om items te beheren.

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

De extension class van de query past logica toe, alleen voor de ``Comment``-resource, zodat de Doctrine query builder alleen reacties toelaat met de status ``published``.

CORS configureren
-----------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Het is standaard niet mogelijk om de API aan te roepen vanaf een ander domein, vanwege het "same-origin" beveiligingsbeleid van moderne HTTP-clients. De CORS-bundle, die geïnstalleerd wordt als deel van ``composer req api``, stuurt Cross-Origin Resource Sharing-headers gebaseerd op de omgevingsvariabele ``CORS_ALLOW_ORIGIN``.

Standaard laat de waarde daarvan, gedefinieerd in ``.env``, HTTP-requests toe vanaf ``localhost`` en ``127.0.0.1``. Dat is precies wat we nodig hebben voor de volgende stap, want we gaan een SPA creëren die een eigen webserver gaat hebben om de API aan te roepen.

.. sidebar:: Verder gaan

    * `SymfonyCasts API Platform tutorial`_;

    * Om de GraphQL-ondersteuning in te schakelen, voer je ``composer require webonyx/graphql-php`` uit en navigeer je vervolgens naar ``/api/graphql``.

.. _`SymfonyCasts API Platform tutorial`: https://symfonycasts.com/screencast/api-platform
