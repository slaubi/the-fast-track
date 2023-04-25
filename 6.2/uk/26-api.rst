Створення API за допомогою API Platform
===========================================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Ми завершили розробку веб-сайту гостьової книги. Тепер, щоб дозволити більш гнучко використовувати дані, як щодо розробки API? API може використовуватися мобільним застосунком, щоб відображати всі конференції, їх коментарі й, можливо, дозволить учасникам відправляти коментарі.

На цьому кроці ми реалізуємо API, що доступний лише для читання.

Встановлення API Platform
-------------------------------------

Можна створити API написавши деякий код, але якщо ми хочемо використовувати стандарти, нам краще використовувати рішення, яке вже бере на себе копітку роботу. Таке рішення, як API Platform:

.. code-block:: terminal

    $ symfony composer req api

Створення API для конференцій
----------------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

Кілька атрибутів у класі Conference — це все, що нам потрібно для налаштування API:

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

Основний атрибут ``ApiResource`` налаштовує API для конференцій. Він обмежує можливі операції методу ``get`` і налаштовує різні речі: наприклад, які поля відображати та в якому порядку.

За замовчуванням основною точкою входу для API є ``/api``, завдяки конфігурації з ``config/routes/api_platform.yaml``, що була додана рецептом пакета.

Веб-інтерфейс дозволяє взаємодіяти з API:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Використовуйте його, щоб перевірити різні можливості:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Уявіть собі, скільки часу буде потрібно, щоб реалізувати все це з нуля!

Створення API для коментарів
--------------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

Зробіть те саме для коментарів:

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

Для налаштування класу використовуються ті самі атрибути.

Обмеження коментарів, що надаються API
--------------------------------------------------------------------

За замовчуванням API Platform надає доступ до всіх записів з бази даних. Але для коментарів тільки опубліковані мають бути частиною API.

Якщо вам потрібно обмежити елементи, що повертаються API, створіть сервіс, який реалізує ``QueryCollectionExtensionInterface``, щоб керувати запитом Doctrine, яка використовується для колекцій, та/або ``QueryItemExtensionInterface``, щоб керувати елементами:

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

Клас розширення запиту застосовує свою логіку тільки до ресурсу ``Comment`` і змінює конструктор запитів Doctrine, щоб враховувати коментарі тільки в стані ``published``.

Налаштування CORS
-----------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

За замовчуванням політика безпеки того ж походження сучасних HTTP-клієнтів робить виклик API з іншого домену забороненим. Бандл CORS, що встановлений як частина ``composer req api``, відправляє заголовки спільного використання ресурсів з різних джерел на основі змінної середовища ``CORS_ALLOW_ORIGIN``.

За замовчуванням його значення, що визначено у ``.env``, дозволяє HTTP-запити від ``localhost`` і ``127.0.0.1`` на будь-який порт. Це саме те, що нам потрібно для наступного кроку, оскільки ми створимо ОЗ у якого буде свій власний веб-сервер, що буде викликати API.

.. sidebar:: Йдемо далі

    * `Навчальний посібник SymfonyCasts: API Platform`_;

    * Щоб увімкнути підтримку GraphQL, виконайте команду ``composer require webonyx/graphql-php``, а потім перейдіть за посиланням ``/api/graphql``.

.. _`Навчальний посібник SymfonyCasts: API Platform`: https://symfonycasts.com/screencast/api-platform
