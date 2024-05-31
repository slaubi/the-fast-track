Описание структуры данных
================================================

.. index::
    single: Doctrine
    single: Database

Для работы с базой данных в PHP мы будем использовать `Doctrine`_ — набор библиотек для управления базами данных: Doctrine DBAL (слой абстракции базы данных), Doctrine ORM (библиотека для манипулирования содержимым нашей базы данных с помощью объектов PHP) и Doctrine Migrations.

Настройка Doctrine ORM
-------------------------------

.. index::
    single: Doctrine;Configuration

Как узнать, подключена ли Doctrine к базе данных? В рецепт Doctrine добавлен конфигурационный файл  ``config/packages/doctrine.yaml``, в котором находятся параметры подключения. Основным параметром в этом файле является *DSN-строка* (Data Source Name — "имя источника данных"), содержащая всю информацию о подключении: учётные данные, хост, порт и т.д. По умолчанию Doctrine ищет переменную  среды ``DATABASE_URL``.

Конфигурация почти всех установленных пакетов находится в директории ``config/packages/``. Как правило, настройки по умолчанию подходят для большинства приложений.

Разбираемся в соглашениях по именованию переменных окружения в Symfony
-----------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Можно инициализировать переменную ``DATABASE_URL`` в файлах ``.env`` или ``.env.local``. Благодаря рецепту пакета, пример значения переменной  ``DATABASE_URL`` уже присутствует в файле ``.env``. Это громоздкое решение, поскольку локальный порт PostgreSQL, открытый через Docker, может измениться. Есть вариант и получше.

Вместо записи переменной ``DATABASE_URL`` в файле, мы можем добавить ко всем командам префикс ``symfony`` и переменная окружения установится автоматически во всех сервисах, запущенных в Docker и/или Platform.sh (при открытом туннеле).

Docker Compose и Platform.sh отлично работают с Symfony благодаря переменным окружения.

.. index::
    single: Symfony CLI;var:export

Проверьте все установленные переменные окружения командой ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Помните *имя сервиса* ``database``, используемое в конфигурациях Docker и Platform.sh? Имена сервисов используются в качестве префиксов для определения переменных окружения, таких как ``DATABASE_URL``. Если ваши сервисы названы в соответствии с соглашениями Symfony, никакой другой конфигурации не требуется.

.. note::

    Базы данных — это не единственный сервис, который использует соглашения Symfony. Например, то же самое можно сказать и о Mailer (через переменную окружения ``MAILER_DSN``).

Изменение начального значения DATABASE_URL в файле .env
----------------------------------------------------------------------------------------

Всё же изменим переменную окружения ``DATABASE_URL`` в файле ``.env``, и укажем, что подключением по умолчанию должен быть PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Почему эту информацию необходимо дублировать в двух разных местах? Потому что в *момент сборки* на некоторых облачных платформах URL-адрес базы данных может быть ещё неизвестен, но Doctrine необходимо знать драйвер базы данных, чтобы создать собственную конфигурацию. Таким образом, хост, имя пользователя и пароль не имеют значения.

Создание классов сущностей
--------------------------------------------------

Класс, описывающий объект конференции со следующими свойствами:

* *city* — город, в котором проводится конференция;

* *year* — год проведения конференции;

* *international* — флаг, указывающий, является ли конференция местной или международной (SymfonyLive или SymfonyCon).

.. index:: ! Command;make:entity

Бандл Maker может помочь нам создать класс (класс *Entity*), представляющий конференцию.

Пришло время создать сущность ``Conference``:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Эта команда является интерактивной: она проведёт вас через процесс добавления всех необходимых полей. Используйте следующие ответы (большинство из них являются ответами по умолчанию, так что вы можете нажать клавишу "Enter", чтобы их использовать):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Полный результат выполнения команды указан ниже:

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

Класс ``Conference`` находится в пространстве имён ``App\Entity\``.

Также команда сгенерировала класс *репозитория* для работы с Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Сгенерированный код выглядит следующим образом (представлена только небольшая часть файла):

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

Обратите внимание, что это обычный PHP-класс без элементов Doctrine. Атрибуты используются для добавления метаданных, позволяющих Doctrine связать класс сущности с соответствующей таблицей в базе данных.

Doctrine добавила свойство ``id`` для хранения первичного ключа строки в таблице базы данных. Этот ключ (``ORM\Id()``) создаётся автоматически (``ORM\GeneratedValue()``) с помощью стратегии, которая зависит от движка базы данных.

.. index::
    single: Command;make:entity

Теперь сгенерируйте класс сущности для комментариев к конференции:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Введите следующие ответы:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``,  ``datetime_immutable``, ``no``.

Связывание сущностей
---------------------------------------

.. index::
    single: Command;make:entity

Обе сущности, конференция и комментарий, должны быть взаимосвязаны. Конференция может иметь ноль или более комментариев, это называется связью *один-ко-многим*.

Используйте команду ``make:entity`` ещё раз, чтобы добавить эту связь в класс ``Conference``:

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

    Если в качестве ответа на вопрос о типе данных вы введёте ``?``, то вы получите список всех поддерживаемых типов:

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

Взгляните на список изменений в классах сущностей после добавления этой связи:

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

Всё, что вам нужно для управления связями, было сгенерировано автоматически. Теперь это ваш код, поэтому не стесняйтесь его изменять, как вам нравится.

Добавление дополнительных свойств
----------------------------------------------------------------

.. index::
    single: Command;make:entity

Я только что понял, что мы забыли добавить одно свойство к сущности комментария: участники, возможно, захотят приложить фотографию с конференции, чтобы наглядно проиллюстрировать про свои отзывы.

Выполните команду ``make:entity`` ещё раз и добавьте свойство/столбец ``photoFilename`` типа ``string`` с возможностью иметь значение ``null``, так как загрузка фотографии не обязательна:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Миграция базы данных
--------------------------------------

.. index:: ! Command;make:migration

Модель проекта теперь полностью описана двумя сгенерированными классами.

Далее нам нужно создать таблицы базы данных, связанные с этими сущностями.

*Doctrine Migrations* идеально подходит для такой задачи. Этот пакет был установлен ранее в виде зависимости для пакета ``orm``.

*Миграция* — это класс, описывающий изменения, необходимые для обновления схемы базы данных с текущего состояния на новое,  определённой в атрибутах сущности. Поскольку на данный момент база данных пуста, миграция должна состоять из создания двух таблиц.

Давайте посмотрим, что сгенерировала Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Обратите внимание на сгенерированное имя файла в выводе командной строки (имя, которое выглядит как ``migrations/Version20191019083640.php``):

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

Обновление локальной базы данных
-------------------------------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Теперь вы можете запустить сгенерированную ранее миграцию для обновления схемы локальной базы данных:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Схема локальной базы данных теперь актуальна и подготовлена для хранения данных.

Обновление базы данных в продакшене
------------------------------------------------------------------

Шаги, необходимые для миграции базы данных на продакшене, такие же, с которыми вы уже знакомы: фиксация изменений и развёртывание.

При развёртывании проекта, Platform.sh не только обновляет код, но и выполняет миграцию базы данных, если таковая имеется (это выясняется при помощи команды ``doctrine:migrations:migrate``).

.. sidebar:: Двигаемся дальше

    * `Базы данных и Doctrine ORM`_ в приложениях Symfony;

    * `Видеокурс по Doctrine на SymfonyCasts`_;

    * `Работа с ассоциациями/связями в Doctrine`_;

    * `Документация DoctrineMigrationsBundle`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Базы данных и Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`Видеокурс по Doctrine на SymfonyCasts`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Работа с ассоциациями/связями в Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`Документация DoctrineMigrationsBundle`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
