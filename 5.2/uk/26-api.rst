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

.. code-block:: bash

    $ symfony composer req api

Створення API для конференцій
----------------------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;Groups

Кілька анотацій у класі Conference — це все, що нам потрібно для налаштування API:

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

Основна анотація ``@ApiResource`` налаштовує API для конференцій. Вона обмежує можливі операції методу ``get`` і налаштовує різні речі: наприклад, які поля відображати та в якому порядку.

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
    single: Annotations;@ApiResource
    single: Annotations;@ApiFilter
    single: Annotations;Groups

Зробіть те саме для коментарів:

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

Для налаштування класу використовуються ті самі анотації.

Обмеження коментарів, що надаються API
--------------------------------------------------------------------

За замовчуванням API Platform надає доступ до всіх записів з бази даних. Але для коментарів тільки опубліковані мають бути частиною API.

Якщо вам потрібно обмежити елементи, що повертаються API, створіть сервіс, який реалізує ``QueryCollectionExtensionInterface``, щоб керувати запитом Doctrine, яка використовується для колекцій, та/або ``QueryItemExtensionInterface``, щоб керувати елементами:

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

Клас розширення запиту застосовує свою логіку тільки до ресурсу ``Comment`` і змінює конструктор запитів Doctrine, щоб враховувати коментарі тільки в стані ``published``.

Налаштування CORS
-----------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

За замовчуванням політика безпеки того ж походження сучасних HTTP-клієнтів робить виклик API з іншого домену забороненим. Бандл CORS, що встановлений як частина ``composer req api``, відправляє заголовки спільного використання ресурсів з різних джерел на основі змінної середовища ``CORS_ALLOW_ORIGIN``.

За замовчуванням його значення, що визначено у ``.env``, дозволяє HTTP-запити від ``localhost`` і ``127.0.0.1`` на будь-який порт. Це саме те, що нам потрібно для наступного кроку, оскільки ми створимо ОЗ у якого буде свій власний веб-сервер, що буде викликати API.

.. sidebar:: Йдемо далі

    * `Навчальний посібник SymfonyCasts: API Platform <https://symfonycasts.com/screencast/api-platform>`_;

    * Щоб увімкнути підтримку GraphQL, виконайте команду ``composer require webonyx/graphql-php``, а потім перейдіть за посиланням ``/api/graphql``.
