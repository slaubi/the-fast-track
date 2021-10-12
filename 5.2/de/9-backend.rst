Ein Admin-Backend einrichten
============================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Das Hinzufügen von bevorstehenden Konferenzen zur Datenbank ist Aufgabe der Projektadministrator*innen. Ein *Admin-Backend* ist ein geschützter Bereich der Website, in dem *Projektadministrator*innen* die Website-Daten verwalten, Feedback-Einsendungen moderieren usw.

Wie können wir das schnell schaffen? Durch die Verwendung eines Bundles, das in der Lage ist, ein Admin-Backend basierend auf dem Modell des Projekts zu generieren. EasyAdmin ist genau das Richtige für Dich.

EasyAdmin konfigurieren
-----------------------

Füge zunächst EasyAdmin als Projektabhängigkeit hinzu:

.. code-block:: bash

    $ symfony composer req "admin:^3"

EasyAdmin generiert automatisch einen Admin-Bereich für Deine Anwendung basierend auf speziellen Controllern. Erstelle ein neues ``src/Controller/Admin/``-Verzeichnis, in dem wir diese Controller speichern werden:

.. code-block:: bash

    $ mkdir src/Controller/Admin/

Zu Beginn lass uns ein "Admin Dashboard" generieren, welches unser Startpunkt sein wird um Website-Daten zu verwalten:

.. code-block:: bash
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

Durch das Akzeptieren der Standardantworten werden die folgenden Controller erstellt:

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

Den Konventionen folgend, sind alle Admin-Controller unter ihrem eigenem ``App\Controller\Admin``-Namespace abgelegt.

Greife auf das generierte Admin-Backend unter ``/admin`` zu. Das ist in der ``index()``-Methode konfiguriert und kann von Dir nach Belieben geändert werden:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Boom! Wir haben ein schönes Gerüst für die Benutzeroberfläche, das nur darauf wartet von Dir angepasst zu werden.

.. index::
    single: CRUD

Der nächste Schritt ist das Erstellen von Controllern um die Konferenzen und Kommentare zu verwalten.

In dem Dashboard-Controller hast Du vielleicht schon die ``configureMenuItems()``-Methode gesehen, welche einen Kommentar hat, wie man Links für "CRUDs" hinzufügt. **CRUD** ist eine Abkürzung für "Create, Read, Update, and Delete" - "Erstellen, Lesen, Aktualisieren und Löschen" - die vier Grundoperationen, die Du für jede Entity haben willst. Das ist genau das, was wir in einem Admin tun wollen; EasyAdmin geht noch einen Schritt weiter und versorgt uns mit einer Suche und Filtern.

Lass uns ein CRUD für Konferenzen erstellen:

.. code-block:: bash
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Wähle ``1`` um ein Admin-Interface für Konferenzen zu erstellen und nutze die Standardwerte für die anderen Fragen. Die folgende Datei sollte generiert worden sein:

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

Mach das selbe für die Kommentare:

.. code-block:: bash
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Zuletzt wollen wir die CRUDs für Konferenzen und Kommentare mit dem Dashboard verlinken:

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

Wir haben die ``configureMenuItems()``-Methode überschrieben um Menüpunkte mit relevanten Icons für Konferenzen und Kommentare und einen Link zurück zur Website-Startseite hinzuzufügen.

EasyAdmin stellt eine API bereit um einfach Entity-CRUDs mittels der ``MenuItem::linkToRoute()``-Methode zu verlinken.

Die Seite mit dem Haupt-Dashboard ist im Moment noch leer. Hier kannst Du ein paar Statistiken oder andere relevante Informationen darstellen. Da wir gerade nichts wichtiges darzustellen haben, leiten wir die Startseite zur Konferenz-Liste um:

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

Um Entity-Relationen (Die passende Konferenz zu dem Kommentar) anzuzeigen, versucht EasyAdmin eine Zeichenketten-Darstellung für die Konferenz zu nutzen. Standardmäßig folgt es der Konvention, die den Entity-Namen und Primärschlüssel nutzt (z. B. ``Conference #1``), wenn die Entity-Klasse keine "magische" ``__toString()``-Methode definiert hat. Füge der ``Conference``-Klasse eine solche Methode hinzu, um die Anzeige lesbarer zu machen:

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

Das Gleiche gilt für die ``Comment``-Klasse:

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

Du kannst Konferenzen nun direkt aus dem Admin-Backend hinzufügen, ändern oder löschen. Spiele damit und füge mindestens eine Konferenz hinzu.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Füge einige Kommentare ohne Fotos hinzu. Stelle das Datum vorerst manuell ein; wir werden die ``createdAt``-Spalte in einem späteren Schritt automatisch ausfüllen.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

EasyAdmin anpassen
------------------

Das Standard-Admin-Backend funktioniert gut, kann aber in vielerlei Hinsicht angepasst werden, um das Nutzungserlebnis zu verbessern. Lass uns einige einfache Änderungen in der ``Comment``-Entity vornehmen, um einige der Möglichkeiten zu demonstrieren:

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

Wir können den Kommentar-Teil anpassen, in dem wir die gewünschten Felder ausdrücklich in der ``configureFields()``-Methode auflisten und so anordnen wie wir wollen. Einige Felder haben ein paar weitere Konfigurationen. Zum Beispiel verstecken wir das Textfeld auf der Index-Seite.

Die ``configureFilters()``-Methode definiert welche Filter oberhalb des Suchfeldes angezeigt werden sollen.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Diese Anpassungen sind nur eine kleine Einführung in die Möglichkeiten von EasyAdmin.

Spiele mit dem Admin, filtere die Kommentare nach Konferenzen oder suche Kommentare z. B. nach E-Mail-Adresse. Das einzige Problem ist, dass jede*r auf das Backend zugreifen kann. Keine Sorge, wir werden es in einem der nächsten Schritt absichern.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Weiterführendes

    * `EasyAdmin-Dokumentation <https://symfony.com/doc/3.x/bundles/EasyAdminBundle/index.html>`_;

    * `Symfony Framework-Konfigurationsreferenz <https://symfony.com/doc/current/reference/configuration/framework.html>`_;

    * `Magische PHP-Methoden <https://www.php.net/manual/en/language.oop5.magic.php>`_.
