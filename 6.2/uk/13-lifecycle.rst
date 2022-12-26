Управління життєвим циклом об'єктів Doctrine
===========================================================================

Створюючи новий коментар, було б чудово, якби дата ``createdAt`` була встановлена автоматично, з використанням значень поточної дати й часу.

В Doctrine є різні способи маніпулювання об'єктами та їх властивостями протягом їх життєвого циклу (до створення запису в базі даних, після оновлення, ...).

Визначення зворотних викликів життєвого циклу
--------------------------------------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

Якщо поведінка не потребує доступу до сервісу й застосовується тільки до одного типу сутності, визначте зворотний виклик в класі сутності:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -91,6 +92,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

*Подія* ``ORM\PrePersist`` оголошується тоді, коли об'єкт вперше зберігається у базі даних. Коли це трапляється, викликається метод ``setCreatedAtValue()`` який використовує поточні дату й час як значення для властивості ``createdAt``.

Додавання "Slugs" для конференцій
--------------------------------------------------------

URL-адреси для конференцій не несуть в собі смислового навантаження: ``/conference/1``. Що ще важливіше, вони розкривають деталі реалізації (витік значення первинного ключа в базі даних).

А як щодо використання URL-адрес на кшталт ``/conference/paris-2020``? Це виглядало б набагато краще. ``paris-2020`` — це те, що ми називаємо *slug* конференції.

.. index::
    single: Command;make:entity

Додайте нову властивість ``slug`` для конференцій (рядок довжиною 255 символів, що не може містити значення ``null``):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Створіть файл міграції, щоб додати новий стовпчик:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

А потім виконайте нову міграцію:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Помилка? Це очікувано. Чому? Хоча ми вказали, що значення "slug" не може містити значення ``null``, під час виконання міграції наявні записи в базі даних конференцій отримають значення ``null``. Виправімо це, змінивши міграцію:

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

Хитрість полягає в тому, щоб додати стовпчик і дозволити йому мати значення ``null``, потім встановити значення, що відмінні від ``null``, і, нарешті, змінити стовпчик "slug" так, щоб не допустити можливості встановлення значення ``null``.

.. note::

    Для реального проекту, використання ``CONCAT(LOWER(city), '-', year)`` може виявитися недостатнім. У цьому випадку нам потрібно було б використовувати "справжній" Slugger.

.. index::
    single: Command;doctrine:migrations:migrate

Тепер міграція має виконатися нормально:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

Оскільки застосунок незабаром використовуватиме "slugs" для пошуку кожної конференції, налаштуймо сутність конференції, щоб мати впевненість, що значення у базі даних будуть унікальними:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
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
    @@ -27,7 +29,7 @@ class Conference
         #[ORM\OneToMany(mappedBy: 'conference', targetEntity: Comment::class, orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(type: 'string', length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

Як ви могли здогадатися, нам потрібно виконати ще один танок з міграціями:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Генерування "Slugs"
------------------------------

.. index::
    single: Components;String
    single: Slug

Генерування "slugs", що добре читаються в URL-адресах (де все, крім символів ASCII, має бути закодовано), є складним завданням, особливо для мов, відмінних від англійської. Наприклад, як конвертувати ``é`` у ``e``?

Замість того щоб винаходити колесо, використовуймо компонент Symfony ``String``, який полегшує маніпуляції з рядками та забезпечує *slugger*.

Додайте метод ``computeSlug()`` до класу ``Conference``, який обчислюватиме значення на основі даних конференції:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -47,6 +48,13 @@ class Conference
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

Метод ``computeSlug()`` обчислює "slug" лише тоді, коли поточне значення порожнє або встановлено спеціальне значення ``-``. Навіщо нам потрібне спеціальне значення ``-``? Тому що при додаванні конференції у панелі керування потрібен "slug". Отже, нам потрібне непорожнє значення, яке повідомляє застосунку про те, що ми хочемо, щоб значення було згенеровано автоматично.

Визначення складних зворотних викликів життєвого циклу
-------------------------------------------------------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

Так само як і для властивості ``createdAt``, значення ``slug`` слід встановлювати автоматично кожного разу, коли конференція оновлюється, викликаючи метод ``computeSlug()``.

Але оскільки цей метод залежить від реалізації ``SluggerInterface``, ми не можемо додати подію ``prePersist``, як і раніше (у нас немає способу для впровадження сервісу slugger).

Натомість створіть слухача сутності Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
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

Зверніть увагу, що значення оновлюється при створенні нової конференції (``prePersist()``) і кожного разу, коли вона оновлюється (``preUpdate()``).

Налаштування сервісу в контейнері
---------------------------------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Досі ми не говорили про один із головних компонентів Symfony — *контейнер впровадження залежностей*. Він керує *сервісами*: створює їх і впроваджує, коли це необхідно.

*Сервіс* — це "глобальний" об'єкт, який надає певні функції (як-от mailer, logger, slugger і т.д.) на відміну від *об'єктів даних* (як-от  екземплярів сутностей Doctrine).

Ви рідко будете працювати з контейнером безпосередньо, оскільки процес впровадження сервісів проходить автоматично всякий раз, коли це необхідно: контейнер впроваджує об'єкти, коли ви вказуєте тип сервісів у якості аргументів контролера.

Якщо ви були здивовані тим, як був зареєстрований слухач подій, на попередньому кроці, тепер у вас є відповідь — завдяки контейнеру. Коли клас реалізує певні інтерфейси, контейнер знає, що клас має бути зареєстровано певним чином.

Оскільки наш клас не реалізує тут інтерфейс і не розширює жодного базового класу, Symfony не знає, як його автоматично налаштувати. Натомість ми можемо використовувати атрибут, щоб сказати контейнеру Symfony, як його підключити:

.. code-block:: diff
    :caption: patch_file

    --- a/src/EntityListener/ConferenceEntityListener.php
    +++ b/src/EntityListener/ConferenceEntityListener.php
    @@ -3,9 +3,13 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\LifecycleEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    Не плутайте слухачів Doctrine і слухачів Symfony. Навіть якщо вони виглядають дуже схожими, вони використовують різну інфраструктуру.

Використання "Slugs" у застосунку
--------------------------------------------------------

Спробуйте додати більше конференцій у панелі керування і змінити місто чи рік наявної; синонім не буде оновлюватися, за винятком тих випадків, коли ви використовуєте спеціальне значення ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

Остання зміна полягає в оновленні контролерів і шаблонів для використання ``slug`` замість ``id`` конференції для маршрутів:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -20,7 +20,7 @@ class ConferenceController extends AbstractController
             ]);
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

Доступ до сторінок конференції тепер має здійснюватися через її "slug":

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Йдемо далі

    * `Система подій Doctrine`_ (зворотні виклики і слухачі життєвого циклу, слухачі сутностей і підписники життєвого циклу);

    * `Документація по компоненту String`_;

    * `Контейнер сервісів`_;

    * `Шпаргалка по сервісах Symfony`_.

.. _`Система подій Doctrine`: https://symfony.com/doc/current/doctrine/events.html
.. _`Документація по компоненту String`: https://symfony.com/doc/current/components/string.html
.. _`Контейнер сервісів`: https://symfony.com/doc/current/service_container.html
.. _`Шпаргалка по сервісах Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
