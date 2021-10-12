Жизненный цикл объектов Doctrine
=====================================================

Было бы неплохо, если при создании нового комментария значение поля ``createdAt`` автоматически заполнялось текущими датой и временем.

Doctrine может по-разному манипулировать объектами и их свойствами в различных стадиях жизненного цикла (до вставки записи в базу данных, после обновления записи и т.д.).

Определение обратных вызовов событий жизненного цикла
-----------------------------------------------------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

Если логика не требует доступа к сервису и применяется только к одному типу сущности, можно определить обратный вызов в классе сущности:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\ORM\Mapping as ORM;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    + * @ORM\HasLifecycleCallbacks()
      */
     class Comment
     {
    @@ -106,6 +107,14 @@ class Comment
             return $this;
         }

    +    /**
    +     * @ORM\PrePersist
    +     */
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTime();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

*Событие* ``@ORM\PrePersist`` срабатывает, когда объект впервые сохранятся в базе данных. В этот момент вызывается метод ``setCreatedAtValue()``, который использует текущие дату и время в качестве значения для свойства ``createdAt``.

Добавление слагов для конференций
---------------------------------------------------------------

Сейчас URL-адреса конференций вроде ``/conference/1`` не очень понятны. Более того, они раскрывают детали реализации приложения (значение первичного ключа ``1`` доступно пользователю).

Почему бы не использовать URL-адреса вида ``/conference/paris-2020``? Они выглядят намного лучше и красивее. Фрагмент адреса ``paris-2020`` — это *слаг* (человеко-понятная часть URL-адреса) конференции.

.. index::
    single: Command;make:entity

Добавьте новое свойство ``slug`` в класс конференции (строка длиной до 255 символов, которая не может пустой):

.. code-block:: bash
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Создайте файл миграции, чтобы добавить новый столбец:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

А затем выполните новую миграцию:

.. code-block:: bash
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Увидели ошибку? Это было ожидаемо, потому что мы указали, что свойство ``slug`` не должно быть пустым (содержать значение ``null``). Но дело в том, что во время выполнения миграции как раз существующие записи в базе данных конференций будут иметь значение ``null``. Давайте исправим это изменив миграцию:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714152808 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema) : void

Мы применили некоторую хитрость: сначала добавляем столбец слага с возможностью иметь значение по умолчанию —``null``, далее создаём слаг для существующих записей (то есть заполняем новый столбец значениями, отличными от ``null``), а затем изменяем столбец слага так, чтобы он он не позволял хранить значение ``null``.

.. note::

    В реальном проекте использование выражения ``CONCAT(LOWER(city), '-', year)`` может быть недостаточным. В таком случае понадобится использовать "настоящий" сервис для генерации слага (слагер).

.. index::
    single: Command;doctrine:migrations:migrate

Теперь миграция должна выполниться без ошибок:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

Поскольку приложение вскоре будет использовать слаги для поиска каждой конференции, давайте улучшим сущность Conference, чтобы гарантировать уникальность значений слагов в базе данных:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,9 +6,11 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    + * @UniqueEntity("slug")
      */
     class Conference
     {
    @@ -40,7 +42,7 @@ class Conference
         private $comments;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, unique=true)
          */
         private $slug;

.. index::
    single: Command;make:migration

Как вы могли догадаться, нам нужно выполнить процедуру миграции:

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Генерация слагов
-------------------------------

.. index::
    single: Components;String
    single: Slug

Во многих языках создать слаг, который хорошо читается в URL-адресе (где должно быть закодировано все, кроме ASCII-символов), не так-то просто. К примеру, как поменять `é`` на ``e``?

Чтобы не изобретать велосипед, давайте воспользуемся Symfony-компонентом ``String``, который не только облегчает работу со строками, но и содержит *слагер*:

.. code-block:: bash

    $ symfony composer req string

В класс ``Conference`` добавьте метод ``computeSlug()``, который исходя из данных конференции сгенерирует слаг:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    @@ -61,6 +62,13 @@ class Conference
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

Метод ``computeSlug()`` генерирует слаг только в том случае, если значение слага отсутствует, либо указанный слаг имеет специальное значение ``-``. Но зачем оно нужно? Поскольку слаг не может быть пустым, при добавлении конференции в административной панели нам нужно указать некое специальное значение (в нашем случае — ``-``) в соответствующем поле, чтобы сообщить приложению, что оно должно автоматически сгенерировать слаг.

Определение сложных обратных вызовов жизненного цикла
-----------------------------------------------------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

По аналогии со свойством ``createdAt``, свойство ``slug`` должно автоматически обновляться при каждом изменении конференции путем вызова метода ``computeSlug()``.

Но так как метод зависит от реализации ``SluggerInterface``, мы не можем добавить событие ``prePersist`` так, как делали это раньше (нет способа внедрить слагер).

Вместо этого создайте обработчик сущности Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        private $slugger;

        public function __construct(SluggerInterface $slugger)
        {
            $this->slugger = $slugger;
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

Обратите внимание, что слаг генерируется как при создании новой конференции (``prePersist()``), так и при её обновлении (``preUpdate()``).

Настройка сервиса в контейнере
---------------------------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

До сих пор мы не упомянули один из главных компонентов Symfony — контейнер внедрения зависимостей, который управляет *сервисами*: создаёт и внедряет их по мере необходимости.

*Сервис* — это "глобальный" объект с определённой функциональностью (mailer — отправка электронных писем, logger — логирование, slugger — генерация URL-адресов, и т.д.) в отличие от *объектов данных* (к примеру, экземпляров сущности Doctrine).

Вы редко будете работать с контейнером напрямую, поскольку он автоматически внедряет сервисы, когда это вам необходимо: внедрение объектов-сервисов происходит, когда вы указываете типы соответствующих сервисов в качестве аргументов контроллера.

Теперь вы знаете, что обработчик события в предыдущем примере был зарегистрирован через контейнер. Когда класс реализует определённые интерфейсы, контейнер знает, что класс должен быть зарегистрирован соответствующим образом.

К сожалению, не всегда такой тип автоматизации возможен, особенно в сторонних пакетах. Одним из таких примеров является только что нами написанный обработчик сущности; он не может автоматически управляться сервисом контейнеров Symfony, так как не реализует ни одного интерфейса и не наследуется от уже "известного" контейнеру класса.

Нам нужно частично объявить обработчик в контейнере. Связывание зависимостей можно пропустить, так как это может быть выполнено контейнером самостоятельно, но требуется вручную добавить пару *тегов*, чтобы зарегистрировать обработчик в диспетчере событий Doctrine:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -29,3 +29,7 @@ services:

         # add more service definitions when explicit configuration is needed
         # please note that last definitions always *replace* previous ones
    +    App\EntityListener\ConferenceEntityListener:
    +        tags:
    +            - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
    +            - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}

.. note::

    Не путайте обработчики событий Doctrine и обработчики событий Symfony. Даже если они очень похожи, они по-разному работают изнутри.

Использование слагов в приложении
---------------------------------------------------------------

Попробуйте добавить несколько конференций в административной панели, либо измените город или год проведения уже созданных конференций; слаг не обновится, только если вы не укажете в его поле специальное значение — ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;@Route

Осталось сделать последнее изменение — заменить в контроллерах и шаблонах параметр маршрутов конференций с ``id` на ``slug``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -31,7 +31,7 @@ class ConferenceController extends AbstractController
         }

         /**
    -     * @Route("/conference/{id}", name="conference")
    +     * @Route("/conference/{slug}", name="conference")
          */
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -10,7 +10,7 @@
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

Теперь можно перейти к странице конференции через её слаг:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Двигаемся дальше

    * `Система событий Doctrine <https://symfony.com/doc/current/doctrine/events.html>`_ (обратные вызовы и обработчики событий жизненного цикла, обработчики сущностей и подписчики жизненного цикла);

    * The `String component docs <https://symfony.com/doc/current/components/string.html>`_;

    * `Сервис-контейнер <https://symfony.com/doc/current/service_container.html>`_;

    * `Шпаргалка по сервисам Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_.
