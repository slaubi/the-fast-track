Налаштування панелі керування
========================================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Додавання майбутніх конференцій до бази даних — завдання адміністраторів проекту. *Панель Керування* — це захищений розділ веб-сайту, де *адміністратори проекту* можуть керувати його вмістом, модерувати відгуки тощо.

Як ми можемо створити це швидко? За допомогою бандла, який здатний згенерувати панель керування на основі моделі проекту. EasyAdmin ідеально підходить для цього.

Встановлення додаткових залежностей
--------------------------------------------------------------------

Навіть якщо пакет ``webapp`` автоматично додав багато чудових пакетів, для деяких більш специфічних функцій нам потрібно додати більше залежностей. Як ми можемо додати більше залежностей? За допомогою Composer. Окрім "звичайних" пакетів Composer, ми будемо працювати з двома "спеціальними" типами пакетів:

* *Компоненти Symfony*: пакети, які реалізують основні функції і низькорівневі абстракції, які потрібні більшості застосунків (маршрутизація, консоль, HTTP-клієнт, розсилка, кеш, ...);

* *Бандли Symfony*: пакети, які додають високорівневі функції чи забезпечують інтеграцію зі сторонніми бібліотеками (пакети здебільшого надаються спільнотою).

Додаймо EasyAdmin як залежність проекту:

.. code-block:: terminal

    $ symfony composer req "easycorp/easyadmin-bundle:^5"

*Псевдоніми* не є функцією Composer, це концепція Symfony для полегшення вашого життя. Псевдоніми є ярликами для популярних пакетів Composer. Хочете ORM для вашого застосунку? Виконайте require ``orm``. Хочете розробити API? Виконайте  require ``api``. Ці псевдоніми автоматично перетворюються в один або кілька звичайних пакетів Composer. Це усвідомлений вибір, зроблений основною командою Symfony.

Ще одна приємна особливість полягає в тому, що ви завжди можете пропустити ім'я постачальника ``symfony``. Виконайте require ``cache`` замість ``symfony/cache``.

.. tip::

    Ви пам'ятаєте, що ми вже згадували плагін Composer під назвою ``symfony/flex``? Псевдоніми є однією з його особливостей.

Налаштування EasyAdmin
----------------------------------

EasyAdmin автоматично генерує панель керування для вашого застосунку на основі конкретних контролерів.

Щоб розпочати роботу з EasyAdmin, створімо "панель керування", яка буде основною точкою входу для управління даними веб-сайту:

.. code-block:: terminal
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

Приймаючи відповіді за замовчуванням, створюється наступний контролер:

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

За домовленістю всі контролери панелі керування зберігаються в їх власному просторі імен ``App\Controller\Admin``.

Отримайте доступ до згенерованої панелі керування за адресою ``/admin``, налаштовану атрибутом ``#[AdminDashboard]``; ви можете змінити URL-адресу на будь-яку, яка вам подобається:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Бум! У нас є приємний інтерфейс панелі керування, готовий бути налаштованим відповідно до наших потреб.

.. index::
    single: CRUD

Наступним кроком є створення контролерів для управління конференціями і коментарями.

У контролері панелі керування ви могли помітити метод ``configureMenuItems()``, який містить коментар щодо додавання посилань на "CRUDs". **CRUD** є абревіатурою від "Create, Read, Update, і Delete", чотирьох основних операцій, які потрібно виконувати над будь-якою сутністю. Це саме те, що ми хочемо, щоб панель керування виконувала для нас; EasyAdmin навіть виводить це на новий рівень, також дбаючи про пошук і фільтрацію.

Створімо CRUD для конференцій:

.. code-block:: terminal
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Виберіть ``1``, щоб створити інтерфейс панелі керування для конференцій і використовуйте значення за замовчуванням для інших питань. Має бути згенеровано наступний файл:

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

Зробіть те саме для коментарів:

.. code-block:: terminal
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Останнім кроком є зв’язування CRUD конференції і коментаря з панеллю керування:

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

Ми перевизначили метод ``configureMenuItems()``, щоб додати пункти меню з відповідними значками конференцій і коментарів, а також додати посилання на головну сторінку веб-сайту. Класи ``ConferenceCrudController`` та ``CommentCrudController`` знаходяться в тому ж просторі імен ``App\Controller\Admin``, що й дашборд, тому не потребують додаткових інструкцій ``use``.

EasyAdmin надає API для полегшення зв'язування з CRUD сутностей за допомогою методу ``MenuItem::linkTo()``, який приймає клас CRUD-контролера.

Головна сторінка панелі керування поки що порожня. Тут ви можете відобразити деяку статистику чи будь-яку відповідну інформацію. Оскільки у нас немає нічого важливого для відображення, переспрямуймо до списку конференцій:

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

При відображенні зв'язків сутності (конференція, пов'язана з коментарем) Easy Admin намагається використовувати рядкове представлення конференції. За замовчуванням він використовує домовленість, яка використовує ім'я сутності і первинний ключ (наприклад, ``Conference #1``), якщо сутність не визначає "магічний" метод ``__toString()``. Щоб зробити відображення змістовнішим, додайте такий метод у клас ``Conference``:

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

Тепер ви можете додавати/змінювати/видаляти конференції безпосередньо з панелі керування. Пограйтеся з цим і додайте хоча б одну конференцію.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Додайте кілька коментарів без фото. Встановіть дату, поки вручну; ми зробимо автоматичне заповнення стовпчика ``createdAt`` на одному з наступних кроків.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Кастомізація EasyAdmin
----------------------------------

Панель керування за замовчуванням працює добре, але її можна налаштувати багатьма способами, щоб поліпшити зручність її використання. Зробімо кілька простих змін у сутності Comment, щоб продемонструвати деякі можливості:

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

Для налаштування розділу ``Comment`` явний перелік полів у методі ``configureFields()`` дозволяє нам упорядкувати їх так, як ми хочемо. Деякі поля налаштовуються додатково, наприклад, приховування текстового поля на індексній сторінці.

Методи ``configureFilters()`` визначають, які фільтри надавати поверх поля пошуку.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Ці налаштування є лише невеликим введенням в можливості, що надаються EasyAdmin.

Пограйтеся з панеллю керування, відфільтруйте коментарі за конференцією чи знайдіть їх, наприклад, за електронною поштою. Єдиний момент — кожен може отримати доступ до панелі керування. Не хвилюйтеся, ми забезпечимо контроль доступу на одному з наступних кроків.

.. code-block:: terminal
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Йдемо далі

    * `Документація по EasyAdmin`_;

    * `Посилання на конфігурацію фреймворку Symfony`_;

    * `Магічні методи PHP`_.

.. _`Документація по EasyAdmin`: https://symfony.com/bundles/EasyAdminBundle/4.x/index.html
.. _`Посилання на конфігурацію фреймворку Symfony`: https://symfony.com/doc/current/reference/configuration/framework.html
.. _`Магічні методи PHP`: https://www.php.net/manual/en/language.oop5.magic.php
