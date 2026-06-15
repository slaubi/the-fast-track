Опис структури даних
======================================

.. index::
    single: Doctrine
    single: Database

Для роботи з базою даних з PHP ми будемо використовувати `Doctrine`_, набір бібліотек, які допомагають розробникам керувати базами даних: Doctrine DBAL (шаблон абстракції бази даних), Doctrine ORM (бібліотека для маніпулювання вмістом нашої бази даних з використанням об'єктів PHP), і Doctrine Migrations.

Налаштування Doctrine ORM
-------------------------------------

.. index::
    single: Doctrine;Configuration

Як Doctrine підключається до бази даних? Рецепт Doctrine додав файл конфігурації ``config/packages/doctrine.yaml``, який містить параметри для підключення. Основним параметром є *DSN бази даних* — рядок, що містить всю інформацію про з'єднання: облікові дані, адреса, порт тощо. За замовчуванням Doctrine шукає змінну середовища ``DATABASE_URL``.

Майже всі встановлені пакети мають конфігурацію в каталозі ``config/packages/``. Значення за замовчуванням здебільшого були ретельно підібрані для роботи в більшості застосунків.

Домовленості про іменування змінних середовища в Symfony
---------------------------------------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Ви можете визначити параметр ``DATABASE_URL`` вручну у файлі ``.env`` або ``.env.local``. Насправді, завдяки рецепту пакета, ви побачите приклад ``DATABASE_URL`` у вашому файлі ``.env``. Але він досить громіздкий, оскільки локальний порт PostgreSQL, що надає нам Docker, може змінюватися. Є кращий спосіб.

Замість того щоб жорстко задати значення ``DATABASE_URL`` у файлі, ми можемо додати префікс ``symfony`` до всіх команд. Це дозволить виявити сервіси запущені у Docker і/чи Platform.sh (коли відкрито тунель) і автоматично встановити змінну середовища.

Docker Compose й Platform.sh легко працюють із Symfony завдяки цим змінним середовища.

.. index::
    single: Symfony CLI;var:export

Перевірте всі доступні змінні середовища, виконавши команду ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Пам'ятаєте *ім'я сервісу* ``database``, що використовується у конфігураціях Docker і Platform.sh? Імена сервісів використовуються як префікси для визначення змінних середовища, таких як ``DATABASE_URL``. Якщо ваші сервіси іменуються відповідно до домовленостей Symfony, то додаткові налаштування не потрібні.

.. note::

    База даних не є єдиним сервісом, що використовує домовленості Symfony. Це ж стосується, наприклад, Mailer (через змінну середовища ``MAILER_DSN``).

Зміна значення DATABASE_URL за замовчуванням у .env
--------------------------------------------------------------------------------

Ми все одно змінимо файл ``.env`` для налаштування значення ``DATABASE_URL`` за замовчуванням, щоб використовувати PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -29,7 +29,7 @@ MESSENGER_TRANSPORT_DSN=doctrine://default?auto_setup=0
     #
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=14&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=14&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Чому інформація має дублюватися у двох різних місцях? Тому, що на деяких хмарних платформах, під *час збірки*, URL-адреса бази даних може бути ще невідома, але Doctrine потрібно знати про систему керування базами даних, щоб створити свою конфігурацію. Таким чином, адреса, ім'я користувача й пароль, насправді, не мають значення.

Створення класів сутностей
--------------------------------------------------

Конференцію можна описати кількома властивостями:

* *Місто*, де організована конференція;

* *Рік* проведення конференції;

* Прапорець *international*, що вказує, чи є конференція локальною або міжнародною (SymfonyLive або SymfonyCon).

.. index:: ! Command;make:entity

Бандл Maker може допомогти нам згенерувати сутність (клас *сутності*), який являє собою конференцію.

Тепер настав час створити сутність ``Conference``:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Ця команда є інтерактивною: вона допоможе вам в процесі додавання всіх необхідних властивостей. Використовуйте наступні відповіді (більшість з них встановлено за замовчуванням, тому ви можете просто натискати клавішу "Enter", щоб застосувати їх):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Ось повний вивід під час виконання команди:

.. code-block:: text
    :emphasize-lines: 8,11,14,17,22,25,28,31,36,39,42,45
    :class: ignore

     created: src/Entity/Conference.php
     created: src/Repository/ConferenceRepository.php

     Entity generated! Now let's add some fields!
     You can always add more fields later manually or by re-running this command.

     New property name (press <return> to stop adding fields):
     > city

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > year

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     > 4

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > isInternational

     Field type (enter ? to see all types) [boolean]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     >



      Success!


     Next: When you're ready, create a migration with make:migration

Клас ``Conference`` знаходиться у просторі імен ``App\Entity\``.

Команда також згенерувала клас *репозиторію* Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Згенерований код виглядає наступним чином (тут наводиться тільки невелика частина файлу):

.. code-block:: php
    :caption: src/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    class Conference
    {
        #[ORM\Column(type: 'integer')]
        #[ORM\Id, ORM\GeneratedValue()]
        private $id;

        #[ORM\Column(type: 'string', length: 255)]
        private $city;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): self
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

Зверніть увагу, що сам клас є простим класом PHP, не пов'язаним з Doctrine. Для додавання метаданих використовуються атрибути, що дозволяють Doctrine зв'язувати клас сутності з відповідною таблицею в базі даних.

Doctrine додала властивість ``id``, щоб зберігати первинний ключ рядка в таблиці бази даних. Цей ключ (``ORM\Id()``) генерується автоматично (``ORM\GeneratedValue()``) за допомогою стратегії, яка залежить від системи керування базою даних.

.. index::
    single: Command;make:entity

Тепер згенеруйте клас сутності для коментарів конференції:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Введіть наступні відповіді:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

Зв'язування сутностей
----------------------------------------

.. index::
    single: Command;make:entity

Дві сутності, конференція та коментар, мають бути зв'язаними між собою. Конференція може мати нуль або більше коментарів, це називається зв'язком *один до багатьох*.

Використовуйте команду ``make:entity`` ще раз, щоб додати цей зв'язок у клас ``Conference``:

.. code-block:: terminal
    :class: answers(comments||OneToMany||Comment||conference||no||yes)

    $ symfony console make:entity Conference

.. code-block:: text
    :emphasize-lines: 4,7,10,15,18,27
    :class: ignore

     Your entity already exists! So let's add some new fields!

     New property name (press <return> to stop adding fields):
     > comments

     Field type (enter ? to see all types) [string]:
     > OneToMany

     What class should this entity be related to?:
     > Comment

     A new property will also be added to the Comment class...

     New field name inside Comment [conference]:
     >

     Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
     > no

     Do you want to activate orphanRemoval on your relationship?
     A Comment is "orphaned" when it is removed from its related Conference.
     e.g. $conference->removeComment($comment)

     NOTE: If a Comment may *change* from one Conference to another, answer "no".

     Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
     > yes

     updated: src/Entity/Conference.php
     updated: src/Entity/Comment.php

.. note::

    Якщо ви введете ``?``, у якості відповіді на питання про тип даних, то отримаєте список всіх підтримуваних типів:

    .. code-block:: text
        :class: ignore

        Main types
          * string
          * text
          * boolean
          * integer (or smallint, bigint)
          * float

        Relationships / Associations
          * relation (a wizard will help you build the relation)
          * ManyToOne
          * OneToMany
          * ManyToMany
          * OneToOne

        Array/Object Types
          * array (or simple_array)
          * json
          * object
          * binary
          * blob

        Date/Time Types
          * datetime (or datetime_immutable)
          * datetimetz (or datetimetz_immutable)
          * date (or date_immutable)
          * time (or time_immutable)
          * dateinterval

        Other Types
          * decimal
          * guid
          * json_array

.. index::
    single: Attributes;ORM\\ManyToOne
    single: Attributes;ORM\\JoinColumn
    single: Attributes;ORM\\OneToMany

Погляньте на повну різницю для класів сутностей після додавання зв'язку:

.. code-block:: diff
    :class: ignore

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -36,6 +36,12 @@ class Comment
          */
         private $createdAt;

    +    #[ORM\ManyToOne(inversedBy: 'comments')]
    +    #[ORM\JoinColumn(nullable: false)]
    +    private Conference $conference;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +94,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): self
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -31,6 +33,16 @@ class Conference
          */
         private $isInternational;

    +    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: "conference", orphanRemoval: true)]
    +    private $comments;
    +
    +    public function __construct()
    +    {
    +        $this->comments = new ArrayCollection();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -71,4 +83,35 @@ class Conference

             return $this;
         }
    +
    +    /**
    +     * @return Collection<int, Comment>
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): self
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments[] = $comment;
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): self
    +    {
    +        if ($this->comments->contains($comment)) {
    +            $this->comments->removeElement($comment);
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

Все, що вам потрібно для управління зв'язком, було згенеровано для вас. Після генерування код стає вашим; сміливо налаштовуйте його так, як вам хочеться.

Додавання додаткових властивостей
----------------------------------------------------------------

.. index::
    single: Command;make:entity

Я щойно зрозумів, що ми забули додати одну властивість до сутності коментаря: відвідувачі, можливо, захочуть долучити фото конференції, щоб проілюструвати свої враження.

Виконайте команду ``make:entity`` ще раз та додайте властивість/стовпчик ``photoFilename`` з типом ``string``, але дозвольте йому мати значення ``null``, оскільки процес завантаження фото є необов'язковим:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Міграція бази даних
------------------------------------

.. index:: ! Command;make:migration

Модель проекту тепер повністю описана двома згенерованими класами.

Далі нам потрібно створити таблиці бази даних, пов'язані з цими сутностями PHP.

*Doctrine Migrations* ідеально підходить для такого завдання. Цей пакет вже встановлено у вигляді залежності для ``orm``.

*Міграція* є класом, що описує зміни, які необхідні для оновлення схеми бази даних із її поточного стану до нового, визначеного в атрибутах сутності. Оскільки база даних поки що порожня, міграція має складатися зі створення двох нових таблиць.

Подивімося, що генерує Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Зверніть увагу на згенероване ім'я файлу у виводі (ім'я, схоже на ``migrations/Version20191019083640.php``)

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version00000000000000 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema): void
        {
            // ...
        }
    }

Оновлення локальної бази даних
---------------------------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Тепер можна запустити згенеровану міграцію, для оновлення схеми локальної бази даних:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Схема локальної бази даних тепер оновлена і готова до зберігання наших даних.

Оновлення бази даних у продакшн
----------------------------------------------------------

Кроки, що необхідні для міграції бази даних у продакшн ті самі, з якими ви вже знайомі: фіксація змін і розгортання.

Під час розгортання проекту Platform.sh оновлює код, але також виконує міграцію бази даних, якщо така є (він виявляє, чи існує команда ``doctrine:migrations:migrate``).

.. sidebar:: Йдемо далі

    * `Бази даних і Doctrine ORM`_ у застосунках Symfony;

    * `Навчальний посібник SymfonyCasts: Doctrine`_;

    * `Робота з асоціаціями/зв'язками Doctrine`_;

    * `Документація по Doctrine Migrations`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Бази даних і Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`Навчальний посібник SymfonyCasts: Doctrine`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Робота з асоціаціями/зв'язками Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`Документація по Doctrine Migrations`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
