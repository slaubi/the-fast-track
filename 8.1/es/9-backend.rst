Configurando un panel de administración
========================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Añadir las próximas conferencias a la base de datos es tarea de los administradores del proyecto. Un *panel de administración* es una sección protegida del sitio web donde *los administradores del proyecto* pueden  gestionar los datos del sitio web, moderar los comentarios y efectuar otro tipo de operaciones.

¿Cómo podemos crearlo rápidamente? Utilizando un *bundle* que  genera un panel de administración a partir del modelo del proyecto. EasyAdmin es perfecto para esta tarea.

Configurando EasyAdmin
----------------------

En primer lugar, añade EasyAdmin como una dependencia del proyecto:

.. code-block:: bash

    $ symfony composer req "admin:^3"

EasyAdmin genera automáticamente un área de administración para tu aplicación basada en controladores específicos. Crea un directorio ``src/Controller/Admin/`` donde se almacenarán estos controladores:

.. code-block:: bash

    $ mkdir src/Controller/Admin/

Para comenzar con EasyAdmin, generaremos un "panel de administración web" que será el punto de entrada principal para gestionar los datos de nuestro sitio web:

.. code-block:: bash
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

Aceptando las respuestas predeterminadas se crea el siguiente controlador:

.. code-block:: php
    :caption: src/Controller/Admin/DashboardController.php
    :class: ignore

    namespace App\Controller\Admin;

    use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
    use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
    use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class DashboardController extends AbstractDashboardController
    {
        /**
         * @Route("/admin", name="admin")
         */
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
            yield MenuItem::linktoDashboard('Dashboard', 'fa fa-home');
            // yield MenuItem::linkToCrud('The Label', 'icon class', EntityClass::class);
        }
    }

Por convención, todos los controladores de administración se almacenan bajo su propio espacio de nombres ``App\Controller\Admin``.

Accede al panel de administración generado en ``/admin`` según lo configurado por el método ``index()``; puedes cambiar la URL a la que quieras:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

¡Boom! Disponemos de una atractiva interfaz shell, lista para ser modificada a nuestras necesidades.

.. index::
    single: CRUD

El siguiente paso es crear los controladores para gestionar conferencias y comentarios.

En el controlador del panel de administración, es posible que te hayas fijado que el método ``configureMenuItems()`` tiene un comentario sobre cómo agregar enlaces a "CRUDs". **CRUD** es un acrónimo de "Crear, Leer, Actualizar y Eliminar", las cuatro operaciones básicas a realizar por cualquier entidad. Eso es exactamente lo que queremos que haga un administrador por nosotros; EasyAdmin incluso lo lleva al siguiente nivel al encargarse también de la búsqueda y filtrado.

Generemos el CRUD para conferencias:

.. code-block:: bash
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Selecciona ``1`` para crear una interfaz de administración para conferencias y usa los valores predeterminados para las otras preguntas. Se debe generar el siguiente archivo:

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

Haz lo mismo para comentarios:

.. code-block:: bash
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

El último paso es vincular los CRUD de administración de conferencia y comentario con el panel de administración:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/DashboardController.php
    +++ b/src/Controller/Admin/DashboardController.php
    @@ -2,6 +2,8 @@

     namespace App\Controller\Admin;

    +use App\Entity\Comment;
    +use App\Entity\Conference;
     use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    @@ -26,7 +28,8 @@ class DashboardController extends AbstractDashboardController

         public function configureMenuItems(): iterable
         {
    -        yield MenuItem::linktoDashboard('Dashboard', 'fa fa-home');
    -        // yield MenuItem::linkToCrud('The Label', 'fas fa-list', EntityClass::class);
    +        yield MenuItem::linktoRoute('Back to the website', 'fas fa-home', 'homepage');
    +        yield MenuItem::linkToCrud('Conferences', 'fas fa-map-marker-alt', Conference::class);
    +        yield MenuItem::linkToCrud('Comments', 'fas fa-comments', Comment::class);
         }
     }

Hemos sobrescrito el método ``configureMenuItems()`` para añadir elementos de menú con iconos relevantes para conferencias, iconos para comentarios y para añadir un enlace de volver a la página de inicio.

EasyAdmin arroja una API para facilitar la vinculación con los CRUDS de la entidad a través del método ``MenuItem::linkToRoute()``.

La página del panel de administración está vacía por ahora. Aquí es donde puedes mostrar algunas estadísticas o cualquier información relevante. Como no tenemos nada importante que mostrar, redirijamos a la lista de conferencias:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/DashboardController.php
    +++ b/src/Controller/Admin/DashboardController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    +use EasyCorp\Bundle\EasyAdminBundle\Router\AdminUrlGenerator;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

    @@ -17,7 +18,10 @@ class DashboardController extends AbstractDashboardController
          */
         public function index(): Response
         {
    -        return parent::index();
    +        $routeBuilder = $this->get(AdminUrlGenerator::class);
    +        $url = $routeBuilder->setController(ConferenceCrudController::class)->generateUrl();
    +
    +        return $this->redirect($url);
         }

         public function configureDashboard(): Dashboard

Cuando se muestran las relaciones entre entidades (la conferencia vinculada a un comentario), EasyAdmin intenta utilizar una representación textual de la conferencia. De manera predeterminada, usa una convención que utiliza el nombre de la entidad y la clave principal (como ``Conferencia #1``) si la entidad no define el método "mágico" ``__toString()``. Para que la visualización sea más significativa, añade dicho método en la clase ``Conferencia``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -44,6 +44,11 @@ class Conference
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

Haz lo mismo para la clase ``Comment``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -48,6 +48,11 @@ class Comment
          */
         private $photoFilename;

    +    public function __toString(): string
    +    {
    +        return (string) $this->getEmail();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

Ahora puedes añadir/modificar/eliminar conferencias directamente desde el panel de administración. Juega con él y añade al menos una conferencia.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Añade algunos comentarios sin fotos. De momento, configura la fecha manualmente; rellenaremos la columna ``createdAt`` automáticamente en un paso posterior.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Personalizando EasyAdmin
------------------------

El panel de administración por defecto funciona bien, pero se puede personalizar de muchas maneras para mejorar la experiencia. Hagamos algunos cambios sencillos en la entidad Comentario para demostrar algunas de sus posibilidades.

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
    @@ -3,7 +3,15 @@
     namespace App\Controller\Admin;

     use App\Entity\Comment;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Crud;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Filters;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
    +use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;

     class CommentCrudController extends AbstractCrudController
     {
    @@ -12,14 +20,44 @@ class CommentCrudController extends AbstractCrudController
             return Comment::class;
         }

    -    /*
    +    public function configureCrud(Crud $crud): Crud
    +    {
    +        return $crud
    +            ->setEntityLabelInSingular('Conference Comment')
    +            ->setEntityLabelInPlural('Conference Comments')
    +            ->setSearchFields(['author', 'text', 'email'])
    +            ->setDefaultSort(['createdAt' => 'DESC']);
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
    +            'html5' => true,
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

Para personalizar la sección ``Comentario``, enumerar los campos explícitamente en el método ``configureFields()`` nos permite ordenarlos del modo deseado. Algunos campos permiten más configuración, como ocultar el campo de texto en la página principal.

El método ``configureFilters()`` define qué filtros serán mostrados sobre del campo de búsqueda.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Estas personalizaciones son solo una pequeña introducción a las posibilidades que ofrece EasyAdmin.

Juega con el panel, filtra los comentarios por conferencia, o busca comentarios por correo electrónico, por ejemplo. Pero todavía nos queda algo por resolver: cualquiera puede acceder al panel de administración. No te preocupes, lo protegeremos más adelante.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Yendo más allá

    * `Documentación de EasyAdmin <https://symfony.com/doc/3.x/bundles/EasyAdminBundle/index.html>`_ ;

    * `Configuración de referencia del *framework* Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_;

    * `PHP métodos mágicos <https://www.php.net/manual/es/language.oop5.magic.php>`_.
