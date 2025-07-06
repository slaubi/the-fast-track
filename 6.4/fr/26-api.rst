Exposer une API avec API Platform
=================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Nous avons terminé la réalisation du site web du livre d'or. Maintenant, pour faciliter l'accès aux données, que diriez-vous d'exposer une API ? Une API pourrait être utilisée par une application mobile pour afficher toutes les conférences, leurs commentaires, et peut-être permettre la soumission de commentaires.

Dans cette étape, nous allons implémenter une API en lecture seule.

Installer API Platform
----------------------

Exposer une API en écrivant du code est possible, mais si nous voulons utiliser des standards, nous ferions mieux d'utiliser une solution qui prend déjà en charge le gros du travail. Une solution comme API Platform :

.. code-block:: terminal

    $ symfony composer req api

Exposer une API pour les conférences
-------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Quelques attributs sur la classe Conference suffisent pour configurer l'API :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -2,29 +2,45 @@

     namespace App\Entity;

    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Attribute\Groups;
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

         /**
    @@ -34,6 +50,7 @@ class Conference
         private Collection $comments;

         #[ORM\Column(length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $slug = null;

         public function __construct()

L'attribut principal ``ApiResource`` configure l'API pour les conférences. Il restreint les opérations possibles à ``get`` et configure différentes choses, comme par exemple, quels champs afficher et comment trier les conférences.

Par défaut, le point d'entrée principal de l'API est ``/api``. Cette configuration a été ajoutée dans ``config/routes/api_platform.yaml`` par la recette du paquet.

Une interface web vous permet d'interagir avec l'API :

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Utilisez-la pour tester les différentes possibilités :

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Imaginez le temps qu'il faudrait pour développer tout cela à partir de zéro !

Exposer une API pour les commentaires
-------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

Faites de même pour les commentaires :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
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
    +use Symfony\Component\Serializer\Attribute\Groups;
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

Le même type d'attributs est utilisé pour configurer la classe.

Filtrer les commentaires exposés par l'API
-------------------------------------------

Par défaut, API Platform expose toutes les entrées de la base de données. Mais pour les commentaires, seuls ceux qui ont été publiés devraient apparaître dans l'API.

Lorsque vous avez besoin de filtrer les éléments retournés par l'API, créez un service qui implémente ``QueryCollectionExtensionInterface`` pour gérer la requête Doctrine utilisée pour les collections, et/ou ``QueryItemExtensionInterface`` pour gérer les éléments :

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

La classe d'extension de requête n'applique sa logique que pour la ressource ``Comment`` et modifie le query builder Doctrine pour ne considérer que les commentaires dans l'état ``published``.

Configurer le CORS
------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Par défaut, la politique de sécurité de même origine des clients HTTP modernes interdit d'appeler l'API depuis un autre domaine. Le paquet CORS, installé par défaut avec ``composer req api``, envoie des en-têtes de *Cross-Origin Resource Sharing* en fonction de la variable d'environnement ``CORS_ALLOW_ORIGIN``.

Par défaut, sa valeur, définie par le fichier ``.env``, autorise les requêtes HTTP depuis ``localhost`` et ``127.0.0.1`` sur n'importe quel port. C'est exactement ce dont nous avons besoin pour la prochaine étape, car nous allons créer une SPA qui aura son propre serveur web et qui appellera l'API.

.. sidebar:: Aller plus loin

    * `Tutoriel SymfonyCasts sur API Platform`_ ;

    * Pour activer la prise en charge de GraphQL, exécutez ``composer require webonyx/graphql-php``, puis accédez à ``/api/graphql``.

.. _`Tutoriel SymfonyCasts sur API Platform`: https://symfonycasts.com/screencast/api-platform
