Esporre un'API con API Platform
===============================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Abbiamo terminato l'implementazione del sito Guestbook. Per consentire una migliore fruizione dei dati, che ne dite di esporre delle API? Un'API potrebbe essere utilizzata da un'applicazione mobile per visualizzare tutte le conferenze, i loro commenti e magari lasciare che i partecipanti inviino commenti.

In questa fase, implementeremo un'API di sola lettura.

Installazione di API Platform
-----------------------------

Esporre un'API scrivendo del codice è possibile, ma se vogliamo usare gli standard è preferibile usare una soluzione che si occupi del lavoro sporco. Una soluzione come API Platform:

.. code-block:: terminal

    $ symfony composer req api

Esposizione di un'API per le conferenze
---------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Qualche attributo sulla classe Conference è tutto ciò di cui abbiamo bisogno per configurare l'API:

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

L'attributo principale ``ApiResource`` configura l'API per le conferenze. Nella fattispecie, limita le operazioni possibili alla sola ``get`` e configura varie cose, quali ad esempio i campi da visualizzare e come ordinare le conferenze.

Per impostazione predefinita, il punto di ingresso principale per l'API è ``/api`` grazie alla configurazione in ``config/routes/api_platform.yaml``,  aggiunta dalla ricetta del pacchetto.

Un'interfaccia web permette di interagire con le API:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Usiamola per testare le varie possibilità:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Immaginate il tempo che ci vorrebbe per fare tutto questo da zero!

Esposizione di un'API per i commenti
------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

Facciamo lo stesso per i commenti:

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

Lo stesso tipo di attributi sono usati per configurare la classe.

Limitare i commenti esposti dall'API
------------------------------------

Per impostazione predefinita, API Platform espone tutte le voci del database. Ma per i commenti, solo quelli pubblicati dovrebbero essere parte dell'API.

Se occorre limitare gli elementi restituiti dall'API, bisogna creare un servizio che implementi ``QueryCollectionExtensionInterface`` per gestire la query usata da Doctrine per reperire le collezioni. In alternativa, si può implementare ``QueryItemExtensionInterface`` per controllare gli elementi:

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

La classe di estensione della query applica la sua logica solo per la risorsa ``Comment`` e modifica il query builder di Doctrine per considerare solo i commenti nello stato ``published``.

Configurare CORS
----------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Per impostazione predefinita, le policy di sicurezza dei moderni client HTTP non consentono la chiamata dell'API da un dominio diverso. NelmioCorsBundle, installato come parte di ``composer req api``, invia gli header Cross-Origin Resource Sharing in base alla variabile d'ambiente ``CORS_ALLOW_ORIGIN``.

Per impostazione predefinita, il suo valore, definito in ``.env``, permette richieste HTTP da ``localhost`` e ``127.0.0.1`` su qualsiasi porta. Questo è esattamente ciò che ci serve, perché nel prossimo passo creeremo una SPA che avrà un suo server web, che richiamerà l'API.

.. sidebar:: Andare oltre

    * `Tutorial API Platform su SymfonyCasts`_;

    * Per abilitare il supporto di GraphQL, eseguire ``composer require webonyx/graphql-php``, quindi visitare l'indirizzo ``/api/graphql``.

.. _`Tutorial API Platform su SymfonyCasts`: https://symfonycasts.com/screencast/api-platform
