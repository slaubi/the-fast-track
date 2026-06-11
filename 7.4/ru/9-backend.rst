Создание административной панели
==============================================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Именно администраторы проекта будут добавлять предстоящие конференции в базу данных. *Административная панель* — это защищённый раздел сайта, где *администраторы проекта* могут изменять данные, модерировать отзывы и многое другое.

Можно быстро сгенерировать панель администрирования на базе модели проекта, используя один из бандлов. EasyAdmin как раз то, что нам нужно.

Установка дополнительных зависимостей
------------------------------------------------------------------------

Пакет ``webapp`` автоматически добавил много разных пакетов, но для нужного функционала нужно добавить больше зависимостей. Как это сделать? С помощью Composer. Помимо "обычных" пакетов Composer, мы будем работать с двумя "специальными" видами пакетов:

* *Symfony Components*: Пакеты, реализующие основные функции и абстракции низкого уровня, которые необходимы большинству приложений (маршрутизация, консоль, HTTP-клиент, почтовый клиент, кеш, ...);

* *Symfony Bundles*: Пакеты, которые добавляют высокоуровневые возможности или обеспечивают интеграцию со сторонними библиотеками (пакеты в основном создаются сообществом).

Давайте добавим бандл EasyAdmin в зависимости проекта:

.. code-block:: terminal

    $ symfony composer req "easycorp/easyadmin-bundle:^5"

*Псевдонимы* — это не функция Composer, а концепция, предоставленная Symfony для облегчения разработки. Псевдонимы — это ярлыки для популярных пакетов Composer. Хотите ORM для вашего приложения? Укажите ``orm``. Хотите разработать API? Укажите ``api``. Псевдонимы автоматически преобразуются в один или несколько обычных пакетов Composer. Это выбор, сделанный основной командой Symfony.

Другая приятная особенность, что вы можете не указывать имя вендора ``symfony``. Укажите ``cache`` вместо ``symfony/cache``.

.. tip::

    Помните, мы уже упоминали о плагине Composer под названием ``symfony/flex``? Псевдонимы — одна из его особенностей.

Настройка бандла EasyAdmin
-----------------------------------------

EasyAdmin автоматически генерирует админ-панель из определённых контроллеров в приложении.

Начнём работу с EasyAdmin с создания "административной панели", которая будет служит главной отправной точкой для управления данными сайта:

.. code-block:: terminal
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

Для создания контролера используйте ответы по умолчанию:

.. code-block:: php
    :caption: src/Controller/Admin/DashboardController.php
    :class: ignore

    namespace App\Controller\Admin;

    use EasyCorp\Bundle\EasyAdminBundle\Attribute\AdminDashboard;
    use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
    use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
    use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    use Symfony\Component\HttpFoundation\Response;

    #[AdminDashboard(routePath: '/admin', routeName: 'admin')]
    class DashboardController extends AbstractDashboardController
    {
        public function index(): Response
        {
            return parent::index();
        }

        public function configureDashboard(): Dashboard
        {
            return Dashboard::new()
                ->setTitle('Guestbook');
        }

        public function configureMenuItems(): iterable
        {
            yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
            // yield MenuItem::linkTo(SomeCrudController::class, 'The Label', 'fas fa-list');
        }
    }

По соглашению все контроллеры, относящиеся к админ-панели, определяются в собственном пространстве имён ``App\Controller\Admin``.

Чтобы посмотреть созданную административную панель в браузере перейдите по пути ``/admin``, который был задан в атрибуте ``#[AdminDashboard]``. Вы можете изменить путь на любой другой:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Вуаля! У нас есть симпатичная административная панель, которую можно настроить как вам угодно.

.. index::
    single: CRUD

В следующем шаге создадим контроллеры для управления конференциями и комментариями.

В контроллере админпанели вы, возможно, обратили внимание на метод ``configureMenuItems()``, в котором закомментирован вызов метода, добавляющего ссылку на "CRUD-действие". **CRUD** — это аббревиатура от "Create, Read, Update и Delete" ("Создать, Прочитать, Обновить, Удалить"), четырёх основных операций, которые можно проделать над любой сущностью. Это именно то, что нам нужно от панели администрирования; Однако этим EasyAdmin не ограничивается — с его помощью ещё можно искать и фильтровать данные.

Давайте сгенерируем CRUD-контроллер для конференций:

.. code-block:: terminal
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Выберите ``1``, чтобы создать контроллер с CRUD-действиями для конференций и используйте значения по умолчанию для других вопросов. Результатом будет следующий файл:

.. code-block:: php
    :caption: src/Controller/Admin/ConferenceCrudController.php
    :class: ignore

    namespace App\Controller\Admin;

    use App\Entity\Conference;
    use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;

    class ConferenceCrudController extends AbstractCrudController
    {
        public static function getEntityFqcn(): string
        {
            return Conference::class;
        }

        /*
        public function configureFields(string $pageName): iterable
        {
            return [
                IdField::new('id'),
                TextField::new('title'),
                TextEditorField::new('description'),
            ];
        }
        */
    }

Сделайте то же самое для комментариев:

.. code-block:: terminal
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Остаётся добавить новые CRUD-интерфейсы конференций и комментариев в админпанель:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/DashboardController.php
    +++ w/src/Controller/Admin/DashboardController.php
    @@ -44,7 +44,8 @@ class DashboardController extends AbstractDashboardController

         public function configureMenuItems(): iterable
         {
    -        yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
    -        // yield MenuItem::linkTo(SomeCrudController::class, 'The Label', 'fas fa-list');
    +        yield MenuItem::linkToRoute('Back to the website', 'fas fa-home', 'homepage');
    +        yield MenuItem::linkTo(ConferenceCrudController::class, 'Conferences', 'fas fa-map-marker-alt');
    +        yield MenuItem::linkTo(CommentCrudController::class, 'Comments', 'fas fa-comments');
         }
     }

Мы переопределили метод ``configureMenuItems()``, в котором добавили пункты меню для конференций и комментариев с соответствующими иконками, а также в самое начало меню поместили ссылку на главную страницу. Классы ``ConferenceCrudController`` и ``CommentCrudController`` находятся в том же пространстве имён ``App\Controller\Admin``, что и панель управления, поэтому дополнительные инструкции ``use`` им не нужны.

С помощью встроенного в EasyAdmin API-метода ``MenuItem::linkTo()``, который принимает класс CRUD-контроллера, легко создавать ссылки на CRUD-сущности.

Главная страница панели пока пуста. В ней вы можете отобразить различную статистику или любую соответствующую информацию. Поскольку у нас нет ничего подобного, то добавим редирект на страницу со списком конференций:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/DashboardController.php
    +++ w/src/Controller/Admin/DashboardController.php
    @@ -8,6 +8,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Attribute\AdminDashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    +use EasyCorp\Bundle\EasyAdminBundle\Router\AdminUrlGenerator;
     use Symfony\Component\HttpFoundation\Response;

     #[AdminDashboard(routePath: '/admin', routeName: 'admin')]
    @@ -15,7 +16,10 @@ class DashboardController extends AbstractDashboardController
     {
         public function index(): Response
         {
    -        return parent::index();
    +        $routeBuilder = $this->container->get(AdminUrlGenerator::class);
    +        $url = $routeBuilder->setController(ConferenceCrudController::class)->generateUrl();
    +
    +        return $this->redirect($url);

             // Option 1. You can make your dashboard redirect to some common page of your backend
             //

При отображении связанных сущностей (в нашем случае, конференции, прикреплённой к комментарию) EasyAdmin попытается преобразовать объект конференции в строку. Если в объекте не будет реализован "магический" метод ``__toString()``, то по умолчанию EasyAdmin выведет имя объекта вместе с первичным ключом (например, ``Conference #1``). Чтобы сделать название связанной сущности более понятнее, определим этот метод в классе ``Conference``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -35,6 +35,11 @@ class Conference
             $this->comments = new ArrayCollection();
         }

    +    public function __toString(): string
    +    {
    +        return $this->city.' '.$this->year;
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

Теперь вы можете добавлять, изменять и удалять конференции непосредственно из административной панели. Изучите его интерфейс и добавьте хотя бы одну конференцию.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Настройка EasyAdmin
----------------------------

Административная панель по умолчанию работает хорошо, хотя она может по-разному настраиваться, чтобы улучшить удобство её использования. Внесем несколько простых изменений в сущность Comment для демонстрации некоторых возможностей:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -3,10 +3,17 @@
     namespace App\Controller\Admin;

     use App\Entity\Comment;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Crud;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Filters;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\IdField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextEditorField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
    +use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;

     class CommentCrudController extends AbstractCrudController
     {
    @@ -15,14 +22,43 @@ class CommentCrudController extends AbstractCrudController
             return Comment::class;
         }

    -    /*
    +    public function configureCrud(Crud $crud): Crud
    +    {
    +        return $crud
    +            ->setEntityLabelInSingular('Conference Comment')
    +            ->setEntityLabelInPlural('Conference Comments')
    +            ->setSearchFields(['author', 'text', 'email'])
    +            ->setDefaultSort(['createdAt' => 'DESC'])
    +        ;
    +    }
    +
    +    public function configureFilters(Filters $filters): Filters
    +    {
    +        return $filters
    +            ->add(EntityFilter::new('conference'))
    +        ;
    +    }
    +
         public function configureFields(string $pageName): iterable
         {
    -        return [
    -            IdField::new('id'),
    -            TextField::new('title'),
    -            TextEditorField::new('description'),
    -        ];
    +        yield AssociationField::new('conference');
    +        yield TextField::new('author');
    +        yield EmailField::new('email');
    +        yield TextareaField::new('text')
    +            ->hideOnIndex()
    +        ;
    +        yield TextField::new('photoFilename')
    +            ->onlyOnIndex()
    +        ;
    +
    +        $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
    +            'years' => range(date('Y'), date('Y') + 5),
    +            'widget' => 'single_text',
    +        ]);
    +        if (Crud::PAGE_EDIT === $pageName) {
    +            yield $createdAt->setFormTypeOption('disabled', true);
    +        } else {
    +            yield $createdAt;
    +        }
         }
    -    */
     }

Чтобы настроить раздел ``Comment``, задайте  и упорядочите поля в методе ``configureFields()``. Некоторые поля настраиваются дополнительно, например, скрытие текстового поля на главной странице.

Добавьте несколько комментариев без фотографий. Пока установите дату вручную, затем в следующих шагах мы сделаем автозаполнение столбца ``createdAt``.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Методы ``configureFilters()`` определяют, какие фильтры будут доступны рядом с полем поиска.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Это всего лишь небольшая часть возможных настроек в EasyAdmin.

Ознакомьтесь с административной панелью, отфильтруйте комментарии по какой-нибудь конференции или, например, найдите их по адресу электронной почты. Однако есть последняя неразрешённая проблема — любой пользователь может войти в панель администрирования. Мы это обязательно исправим в следующих шагах.

.. code-block:: terminal
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Двигаемся дальше

    * `Документация EasyAdmin`_;

    * `Справочник по конфигурированию Symfony`_;

    * `Магические методы в PHP`_.

.. _`Документация EasyAdmin`: https://symfony.com/bundles/EasyAdminBundle/4.x/index.html
.. _`Справочник по конфигурированию Symfony`: https://symfony.com/doc/current/reference/configuration/framework.html
.. _`Магические методы в PHP`: https://www.php.net/manual/en/language.oop5.magic.php
