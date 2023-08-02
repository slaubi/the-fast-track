Udostępnianie API za pomocą biblioteki API Platform
=====================================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Zakończyliśmy implementację strony internetowej księgi gości. Aby lepiej wykorzystać nasze dane, może udostępnimy teraz API? API może być używane przez aplikację mobilną do wyświetlania wszystkich konferencji, ich komentarzy, a może nawet umożliwić uczestnikom dodawanie nowych komentarzy.

W tym kroku zamierzamy wdrożyć API tylko do odczytu.

Instalowanie API Platform
-------------------------

Udostępnianie API poprzez napisanie kodu samemu jest możliwe, ale jeśli zależy nam na zgodności ze standardami, lepiej użyć rozwiązania, które wykona za nas tę ciężką pracę. Rozwiązanie takie jak API Platform:

.. code-block:: terminal

    $ symfony composer req api

Udostępnianie API dla konferencji
----------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Kilka atrybutów w klasie Conference to wszystko, czego potrzebujemy, aby skonfigurować API:

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

         #[ORM\Column(length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $slug = null;

         public function __construct()

Główny atrybut ``ApiResource`` konfiguruje API dla konferencji. Ogranicza możliwe operacje do ``get``, ustawia sposób sortowania listy konferencji i wskazuje, jakie pola wyświetlać.

Domyślnie, głównym punktem wejścia dla API jest ``/api`` dzięki konfiguracji w ``config/routes/api_platform.yaml`` która została dodana przez przepis (ang. recipe) pakietu.

Interfejs webowy pozwala na interakcję z API:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Użyj go, aby przetestować różne możliwości:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Wyobraź sobie, ile czasu zajęłoby wdrożenie tego wszystkiego od zera!

Udostępnienie API dla komentarzy
---------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

Zrób to samo w przypadku komentarzy:

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

Ten sam rodzaj atrybutów jest używany do konfiguracji klasy.

Ograniczanie komentarzy udostępnionych przez API
-------------------------------------------------

Domyślnie, API Platform udostępnia wszystkie wpisy z bazy danych. Ale w przypadku komentarzy, API powinno zwracać tylko te opublikowane.

Gdy musisz ograniczyć liczbę elementów zwracanych przez API, utwórz usługę (ang. service), która przejmie kontrolę nad zapytaniem używanym przez Doctrine dla kolekcji, jeśli zaimplementujesz ``QueryCollectionExtensionInterface``, i/lub poszczególnych elementów, jeśli zaimplementujesz``QueryItemExtensionInterface``.

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
        public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }
    }

Klasa rozszerzeń zapytań stosuje swój schemat działania tylko w odniesieniu do zasobów ``Comment`` i modyfikuje konstruktor zapytań Doctrine (ang. Doctrine query builder), aby uwzględnić tylko komentarze oznaczone jako ``published``.

Konfigurowanie CORS
-------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Domyślnie reguła tego samego pochodzenia (ang. same-origin security policy) nowoczesnych klientów HTTP sprawia, że wywoływanie API z innej domeny jest zabronione. Pakiet CORS, zainstalowany jako część ``composer req api``, wysyła nagłówki Cross-Origin Resource Sharing oparte na zmiennej środowiskowej ``CORS_ALLOW_ORIGIN``.

Domyślnie, jego wartość zdefiniowana w ``.env`` pozwala na odbieranie żądań HTTP z ``localhost`` i ``127.0.0.1`` na dowolnym porcie. To jest dokładnie to, czego potrzebujemy w kolejnym kroku, ponieważ stworzymy aplikację jednostronicową (ang. single-page application, SPA) mającą swój własny serwer WWW, która będzie odwoływać się do API.

.. sidebar:: Idąc dalej

    * `Samouczek SymfonyCasts dotyczący API Platform`_;

    * Aby włączyć obsługę GraphQL, uruchom ``composer require webonyx/graphql-php``, a następnie przejdź do ``/api/graphql``.

.. _`Samouczek SymfonyCasts dotyczący API Platform`: https://symfonycasts.com/screencast/api-platform
