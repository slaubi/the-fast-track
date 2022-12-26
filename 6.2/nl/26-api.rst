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
    @@ -2,35 +2,52 @@

     namespace App\Entity;

    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
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
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'conference:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'conference:list'])
    +    ],
    +    order: ['year' => 'DESC', 'city' => 'ASC'],
    +    paginationEnabled: false,
    +)]
     class Conference
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $city = null;

         #[ORM\Column(length: 4)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $year = null;

         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?bool $isInternational = null;

         #[ORM\OneToMany(mappedBy: 'conference', targetEntity: Comment::class, orphanRemoval: true)]
         private Collection $comments;

         #[ORM\Column(type: 'string', length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $slug = null;

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
    @@ -2,41 +2,63 @@

     namespace App\Entity;

    +use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
    +use ApiPlatform\Metadata\ApiFilter;
    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\CommentRepository;
     use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    +#[ApiResource(
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'comment:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'comment:list'])
    +    ],
    +    order: ['createdAt' => 'DESC'],
    +    paginationEnabled: false,
    +)]
    +#[ApiFilter(SearchFilter::class, properties: ['conference' => 'exact'])]
     class Comment
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $author = null;

         #[ORM\Column(type: Types::TEXT)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $text = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $email = null;

         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?\DateTimeImmutable $createdAt = null;

         #[ORM\ManyToOne(inversedBy: 'comments')]
         #[ORM\JoinColumn(nullable: false)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?Conference $conference = null;

         #[ORM\Column(length: 255, nullable: true)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $photoFilename = null;

         #[ORM\Column(length: 255, options: ['default' => 'submitted'])]

Dezelfde soort attributen worden gebruikt om de class te configureren.

API restricties opleggen bij de reacties
----------------------------------------

Standaard geeft API Platform alle gegevens uit de database vrij. Maar eigenlijk moeten alleen de gepubliceerde reacties deel van de API zijn.

Om te beperken welke items door de API worden teruggestuurd, maak je een service aan die de ``QueryCollectionExtensionInterface`` implementeert, om de Doctrine-query te beheren die gebruikt wordt voor collections, en/of de ``QueryItemExtensionInterface``, om items te beheren.

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 14-16,21-23

    namespace App\Api;

    use ApiPlatform\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use ApiPlatform\Metadata\Operation;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator,     string $resourceClass, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string     $resourceClass, array $identifiers, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
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
