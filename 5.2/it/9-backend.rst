Impostare un pannello amministrativo
====================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Aggiungere le prossime conferenze al database è compito degli amministratori del progetto. Il pannello amministrativo è una sezione protetta del sito web dove gli *amministratori di progetto* possono gestire i dati del sito, moderare i feedback e altro ancora.

Come possiamo crearlo in fretta? Usando un bundle che è in grado di generare un pannello amministrativo basato sul modello del progetto. EasyAdmin si adatta perfettamente al contesto.

Configurare EasyAdmin
---------------------

Per iniziare, aggiungiamo EasyAdmin come dipendenza del progetto:

.. code-block:: bash

    $ symfony composer req "admin:^3"

EasyAdmin genera automaticamente un'area di amministrazione per l'applicazione, in base a specifici controller. Creiamo una nuova cartella ``src/Controller/Admin/``, dove salvare questi controller:

.. code-block:: bash

    $ mkdir src/Controller/Admin/

Per iniziare con EasyAdmin, generiamo una "web admin dashboard", che sarà il punto di ingresso principale per la gestione dei dati del sito:

.. code-block:: bash
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

Accettare le risposte predefinite per creare il seguente controller:

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

Per convenzione, tutti i controller di amministrazione sono sotto il namespace ``App\Controller\Admin``.

Accedere al pannello amministrativo generato in ``/admin``, come configurato nel metodo ``index()``. Si può modificare l'URL a piacimento:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Fatto! Abbiamo un'interfaccia di amministrazione bella e ricca di funzionalità, pronta per essere personalizzata.

.. index::
    single: CRUD

Il passo successivo è quello di creare i controller per gestire le conferenze e i commenti.

Potreste aver notato il metodo ``configureMenuItems()`` nel controller principale, con un commento che parla di aggiungere collegamenti ai "CRUD": **CRUD** è un acronimo per "Create, Read, Update, Delete" (Creare, Leggere, Aggiornare, Eliminare), le quattro operazioni di base che si possono eseguire su un'entità. Questo è esattamente ciò che vogliamo da un'interfaccia di amministrazione. EasyAdmin si occupa anche di ricerca e filtri.

Generiamo un CRUD per le conferenze:

.. code-block:: bash
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

Scegliamo ``1`` per creare un'interfaccia di amministrazione per le conferenze e lasciamo le altre risposte ai valori predefiniti. Dovrebbe generarsi il seguente file:

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

Facciamo la stessa cosa per i commenti:

.. code-block:: bash
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

L'ultimo passo è quello di collegare alla dashboard i CRUD di amministrazione per conferenze e commenti

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

Abbiamo sovrascritto il metodo ``configureMenuItems`` per aggiungere elementi di menù con le icone pertinenti a conferenze e commenti, e per aggiungere un link di ritorno alla home page.

EasyAdmin espone una API per facilitare il collegamento dei CRUD delle entità tramite il metodo ``MenuItem::linkToRoute()``

Per il momento la dashboard della pagina principale è vuota. In questa pagina si potranno mostrare statistiche o altre informazioni d'interesse. Siccome non abbiamo niente di importante da mostrare, facciamo un redirect alla lista delle conferenze:

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

Quando si mostrano le relazioni tra entità (la conferenza relativa a un commento), EasyAdmin cerca di rappresentare una conferenza come stringa. Come strategia predefinita, utilizza una convenzione composta dalla concatenazione del nome dell'entità e la sua chiave primaria (come ad esempio ``Conference #1``) se l'entità non ha definito il metodo "magico" ``__toString()``. Per rendere questo valore più significativo, aggiungiamo il suddetto metodo alla classe ``Conference``:

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

Fare lo stesso per la classe ``Comment``:

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

Ora è possibile aggiungere/modificare/cancellare le conferenze direttamente dal pannello amministrativo. Si può fare qualche prova e aggiungere almeno una conferenza.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

Aggiungiamo qualche commento senza foto e impostiamo la data manualmente: la colonna ``createdAt`` sarà automatizzata in un secondo momento.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Personalizzazione di EasyAdmin
------------------------------

Il pannello amministrativo predefinito funziona bene, ma può essere personalizzato in molti modi per migliorare l'esperienza utente. Facciamo alcune semplici modifiche per dimostrarne le possibilità. Modificare la configurazione corrente con quanto segue:

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

Per personalizzare la sezione ``Comment``, elencare i campi esplicitamente nel metodo ``configureFields()`` ci consentirà di ordinarli nella maniera che desideriamo. Alcuni campi sono configurati ulteriormente, come il nascondere il campo testuale nella pagina di partenza.

I metodi ``configureFilters()`` definiscono quali filtri esporre al di sopra del campo di ricerca.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

Queste personalizzazioni sono solo una piccola introduzione alle possibilità offerte da EasyAdmin.

Giocate con il pannello amministrativo, filtrando i commenti per conferenza o cercando i commenti per e-mail, ad esempio. L'unico problema è che chiunque può accedere al backend. Niente paura, l'accesso sarà regolato in un secondo momento.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Andare oltre

    * `Documentazione di EasyAdmin <https://symfony.com/doc/3.x/bundles/EasyAdminBundle/index.html>`_;

    * `Riferimento alla configurazione del framework Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_;

    * `Metodi magici di PHP <https://www.php.net/manual/en/language.oop5.magic.php>`_.
