Feedback ontvangen via formulieren
==================================

.. index::
    single: Components;Form
    single: Form

Tijd om onze deelnemers feedback te laten geven op conferenties. Ze zullen hun reactie indienen via een *HTML-formulier*.

Form types genereren
--------------------

.. index::
    single: Command;make:form

Gebruik de Maker bundle om een form class te genereren:

.. code-block:: terminal

    $ symfony console make:form CommentType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

De ``App\Form\CommentType`` class definieert een form voor de ``App\Entity\Comment`` entity:

.. code-block:: php
    :caption: src/Form/CommentType.php
    :class: ignore

    namespace App\Form;

    use App\Entity\Comment;
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\Component\OptionsResolver\OptionsResolver;

    class CommentType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('author')
                ->add('text')
                ->add('email')
                ->add('createdAt')
                ->add('photoFilename')
                ->add('conference')
            ;
        }

        public function configureOptions(OptionsResolver $resolver): void
        {
            $resolver->setDefaults([
                'data_class' => Comment::class,
            ]);
        }
    }

Een `form type`_ beschrijft de *formuliervelden* die aan een model gebonden zijn. Het voert dataconversie uit tussen de ingediende gegevens en de properties van de model class. Standaard gebruikt Symfony metadata - zoals de Doctrine metadata - van de ``Comment`` entity om de configuratie van elk veld te raden. Het ``text`` veld wordt bijvoorbeeld weergegeven als een veld ``textarea`` omdat het een grotere kolom in de database gebruikt.

Een formulier weergeven
-----------------------

Om een formulier weer te geven maak je het aan in de controller en geeft je het door aan de template:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 20,30

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,8 +2,10 @@

     namespace App\Controller;

    +use App\Entity\Comment;
     use App\Entity\Conference;
    +use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    @@ -23,6 +25,9 @@ final class ConferenceController extends AbstractController
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
    +        $comment = new Comment();
    +        $form = $this->createForm(CommentType::class, $comment);
    +
             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    @@ -31,6 +36,7 @@ final class ConferenceController extends AbstractController
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    +            'comment_form' => $form,
             ]);
         }
     }

Je moet nooit direct het formuliertype instantiëren. Gebruik in plaats daarvan de ``createForm()`` methode. Deze methode maakt deel uit van ``AbstractController`` en vergemakkelijkt het maken van formulieren.

.. index::
    single: Twig;form

Het formulier weergeven in een template kan je met de ``form`` Twig functie:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 10

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -30,4 +30,8 @@
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}
    +
    +    <h2>Add your own feedback</h2>
    +
    +    {{ form(comment_form) }}
     {% endblock %}

Bij het vernieuwen van een conferentiepagina in de browser, merk je op dat elk formulierveld de juiste HTML-widget toont (het gegevenstype is afgeleid van het model):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

De ``form()`` functie genereert het HTML-formulier op basis van alle informatie die in het formuliertype gedefinieerd is. Het voegt ook ``enctype=multipart/form-data`` toe aan de ``<form>`` tag, zoals dat nodig is voor het invoerveld voor het uploaden van bestanden. Bovendien zorgt de functie er voor dat er foutmeldingen worden weergegeven wanneer de inzending foute gegevens bevat. Alles kan worden aangepast door de standaard templates te overschrijven, maar dat hebben we voor dit project niet nodig.

Een Form Type aanpassen
-----------------------

Zelfs als formuliervelden worden geconfigureerd op basis van hun model-class, kan je de standaard configuratie in de Form type class rechtstreeks aanpassen:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Form/CommentType.php
    +++ w/src/Form/CommentType.php
    @@ -6,26 +6,32 @@ use App\Entity\Comment;
     use App\Entity\Conference;
     use Symfony\Bridge\Doctrine\Form\Type\EntityType;
     use Symfony\Component\Form\AbstractType;
    +use Symfony\Component\Form\Extension\Core\Type\EmailType;
    +use Symfony\Component\Form\Extension\Core\Type\FileType;
    +use Symfony\Component\Form\Extension\Core\Type\SubmitType;
     use Symfony\Component\Form\FormBuilderInterface;
     use Symfony\Component\OptionsResolver\OptionsResolver;
    +use Symfony\Component\Validator\Constraints\Image;

     class CommentType extends AbstractType
     {
         public function buildForm(FormBuilderInterface $builder, array $options): void
         {
             $builder
    -            ->add('author')
    +            ->add('author', null, [
    +                'label' => 'Your name',
    +            ])
                 ->add('text')
    -            ->add('email')
    -            ->add('createdAt', null, [
    -                'widget' => 'single_text',
    +            ->add('email', EmailType::class)
    +            ->add('photo', FileType::class, [
    +                'required' => false,
    +                'mapped' => false,
    +                'constraints' => [
    +                    new Image(maxSize: '1024k')
    +                ],
                 ])
    -            ->add('photoFilename')
    -            ->add('conference', EntityType::class, [
    -                'class' => Conference::class,
    -                'choice_label' => 'id',
    -            ])
    -        ;
    +            ->add('submit', SubmitType::class)
    +       ;
         }

         public function configureOptions(OptionsResolver $resolver): void

Merk op dat we een submit knop hebben toegevoegd (maar we nog steeds de ``{{ form(comment_form) }}`` aanroep kunnen blijven gebruiken in de template).

Sommige velden kunnen niet automatisch worden geconfigureerd, zoals het ``photoFilename`` veld. De ``Comment`` entity hoeft alleen de bestandsnaam van de foto op te slaan, maar het formulier moet wel het uploaden van het bestand afhandelen. Om dit op te lossen hebben we een veld ``photo`` toegevoegd als niet-``gemapped`` veld. Het wordt niet toegevoegd als property op de ``Comment`` entity. We zullen het handmatig instellen om specifieke logica te implementeren (zoals het opslaan van de geüploade foto op de schijf).

Om een voorbeeld te geven van aanpassingen hebben we ook het standaardlabel van enkele velden aangepast.

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Modellen valideren
------------------

Het Form Type configureert de frontend rendering van het formulier (via HTML5 validatie). Hier is het gegenereerde HTML-formulier:

.. code-block:: html
    :class: ignore

    <form name="comment_form" method="post" enctype="multipart/form-data">
        <div id="comment_form">
            <div >
                <label for="comment_form_author" class="required">Your name</label>
                <input type="text" id="comment_form_author" name="comment_form[author]" required="required" maxlength="255" />
            </div>
            <div >
                <label for="comment_form_text" class="required">Text</label>
                <textarea id="comment_form_text" name="comment_form[text]" required="required"></textarea>
            </div>
            <div >
                <label for="comment_form_email" class="required">Email</label>
                <input type="email" id="comment_form_email" name="comment_form[email]" required="required" />
            </div>
            <div >
                <label for="comment_form_photo">Photo</label>
                <input type="file" id="comment_form_photo" name="comment_form[photo]" />
            </div>
            <div >
                <button type="submit" id="comment_form_submit" name="comment_form[submit]">Submit</button>
            </div>
            <input type="hidden" id="comment_form__token" name="comment_form[_token]" value="DwqsEanxc48jofxsqbGBVLQBqlVJ_Tg4u9-BL1Hjgac" />
        </div>
    </form>

Het formulier maakt gebruik van de ``email`` invoer voor de reactie e-mail, en maakt het grootste deel van de velden ``required``. Merk op dat het formulier ook een verborgen ``_token`` veld bevat om het formulier te beschermen tegen `CSRF-aanvallen`_.

Maar als het indienen van het formulier de HTML-validatie omzeilt (door gebruik te maken van een HTTP client die deze validatieregels niet afdwingt, zoals cURL), kunnen ongeldige gegevens op de server terecht komen.

We moeten ook een aantal validaties toevoegen aan het ``Comment`` datamodel:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -5,6 +5,7 @@ namespace App\Entity;
     use App\Repository\CommentRepository;
     use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    @@ -16,12 +17,16 @@ class Comment
         private ?int $id = null;

         #[ORM\Column(length: 255)]
    +    #[Assert\NotBlank]
         private ?string $author = null;

         #[ORM\Column(type: Types::TEXT)]
    +    #[Assert\NotBlank]
         private ?string $text = null;

         #[ORM\Column(length: 255)]
    +    #[Assert\NotBlank]
    +    #[Assert\Email]
         private ?string $email = null;

         #[ORM\Column]

Formulieren afhandelen
----------------------

We hebben voldoende code geschreven om het formulier weer te geven.

We moeten het indienen van het formulier en het opslaan in de database afhandelen in de controller:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,8 +7,10 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
    @@ -14,6 +15,11 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    +    public function __construct(
    +        private EntityManagerInterface $entityManager,
    +    ) {
    +    }
    +
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    @@ -24,10 +30,19 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    public function show(Request $request, #[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentType::class, $comment);
    +        $form->handleRequest($request);
    +        if ($form->isSubmitted() && $form->isValid()) {
    +            $comment->setConference($conference);
    +
    +            $this->entityManager->persist($comment);
    +            $this->entityManager->flush();
    +
    +            return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
    +        }

             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

Wanneer het formulier wordt ingediend, wordt het ``Comment``-object bijgewerkt aan de hand van de ingediende gegevens.

De conferentie moet dezelfde zijn als die van de URL (we hebben deze uit het formulier verwijderd).

Als het formulier niet geldig is, tonen we de pagina, maar het formulier zal nu de ingediende waarden en foutmeldingen bevatten, zodat deze aan de gebruiker kunnen worden getoond.

Test het formulier. Het zou moeten werken en de gegevens zouden opgeslagen moeten zijn in de database (controleer dit in de admin backend). Er is echter één probleem: foto's. Deze werken niet omdat we ze nog niet in de controller afhandelen.

Bestanden uploaden
------------------

Geüploade foto's moeten opgeslagen worden op de lokale schijf, op een plaats die toegankelijk is via de frontend, zodat we ze kunnen weergeven op de conferentie-pagina. We gebruiken de map ``public/uploads/photos`` hiervoor.

.. index::
    single: Attribute;Autowire
    single: Autowire

Omdat we het directory pad niet willen hardcoden, hebben we een manier nodig om deze globaal in de configuratie op te slaan. De Symfony Container heeft de mogelijkheid om, naast services, ook *parameters* op te slaan. Dit zijn scalars die services helpen configureren:

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -4,6 +4,7 @@
     # Put parameters here that don't need to change on each machine where the app is deployed
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
    +    photo_dir: "%kernel.project_dir%/public/uploads/photos"

     services:
         # default configuration for services in *this* file

We hebben al gezien hoe services automatisch geïnjecteerd worden via constructor argumenten. Container parameters kunnen we expliciet injecteren middels de ``Autowire`` attribute.

Nu weten we alles dat we nodig hebben om de logica voor het opslaan van het geuploadde bestand te kunnen implementeren:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,7 +9,8 @@ use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    @@ -29,13 +30,24 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, #[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    -    {
    +    public function show(
    +        Request $request,
    +        #[MapEntity(mapping: ['slug' => 'slug'])]
    +        Conference $conference,
    +        CommentRepository $commentRepository,
    +        #[Autowire('%photo_dir%')] string $photoDir,
    +        #[MapQueryParameter] int $offset = 0,
    +    ): Response {
             $comment = new Comment();
             $form = $this->createForm(CommentType::class, $comment);
             $form->handleRequest($request);
             if ($form->isSubmitted() && $form->isValid()) {
                 $comment->setConference($conference);
    +            if ($photo = $form['photo']->getData()) {
    +                $filename = bin2hex(random_bytes(6)).'.'.$photo->guessExtension();
    +                $photo->move($photoDir, $filename);
    +                $comment->setPhotoFilename($filename);
    +            }

                 $this->entityManager->persist($comment);
                 $this->entityManager->flush();

Bij het afhandelen van foto-uploads maken we een willekeurige naam voor het bestand. Vervolgens verplaatsen we het geüploade bestand naar de uiteindelijke locatie (de foto map). Tenslotte slaan we de bestandsnaam op in het Comment-object.

Probeer een PDF-bestand te uploaden in plaats van een foto. Je zal een foutmeldingen te zien krijgen. Het design is op dit moment nogal lelijk, maar maak je geen zorgen, alles zal in de volgende stappen mooier worden, wanneer we aan het design van de website gaan werken. Om styling te geven aan alle form elements zullen we één regel configuratie wijzigen.

Formulieren debuggen
--------------------

Wanneer een formulier wordt ingediend en iets werkt niet goed, gebruik dan het "Form" scherm van de Symfony Profiler. Het geeft je informatie over het formulier, alle opties, de ingediende gegevens en hoe deze intern worden omgezet. Als het formulier fouten bevat, worden deze ook vermeld.

De typische formulier-workflow gaat als volgt:

* Het formulier wordt weergegeven op een pagina;

* De gebruiker dient het formulier in via een POST request;

* De server leidt de gebruiker om naar een andere pagina of naar dezelfde pagina.

Maar hoe kun je toegang krijgen tot de profiler bij een succesvol request? Omdat de pagina onmiddellijk wordt omgeleid, krijgen we de web debug toolbar voor het POST-request nooit te zien. Geen probleem: beweeg je muis over het linker, groene "200" gedeelte op de omgeleide pagina. Je zou hier de "302" omleiding moeten zien met een link naar het profile (tussen haakjes).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Klik erop om toegang te krijgen tot het POST-requestprofile en ga dan naar het "Form" scherm:

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Geüploade foto's weergeven in de admin backend
-----------------------------------------------

De admin backend toont op dit moment de bestandsnaam van de foto, maar we willen de daadwerkelijke foto zien:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -10,6 +10,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\IdField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\ImageField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextEditorField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
    @@ -47,7 +48,9 @@ class CommentCrudController extends AbstractCrudController
             yield TextareaField::new('text')
                 ->hideOnIndex()
             ;
    -        yield TextField::new('photoFilename')
    +        yield ImageField::new('photoFilename')
    +            ->setBasePath('/uploads/photos')
    +            ->setLabel('Photo')
                 ->onlyOnIndex()
             ;

Geüploade foto's niet in Git opnemen
-------------------------------------

Commit dit nog niet! We willen namelijk geen geüploade afbeeldingen opslaan in de Git repository. Voeg de ``/public/uploads`` map toe aan het ``.gitignore`` bestand:

.. code-block:: diff
    :caption: patch_file

    --- i/.gitignore
    +++ w/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Het opslaan van geüploade bestanden op productieservers
--------------------------------------------------------

De laatste stap is het opslaan van de geüploade bestanden op productieservers. Maar waarom moeten we hier iets speciaals doen? Moderne cloud-platforms maken vaak gebruik van alleen-lezen containers en Platform.sh is geen uitzondering op deze regel.

Niet alles is alleen-lezen in een Symfony project. Symfony doet zijn best om zoveel mogelijk cache te genereren bij het opbouwen van de container (tijdens de cache warmup fase), maar Symfony moet nog steeds de gebruikerscache kunnen schrijven, de logs, de sessies (als ze op het filesystem worden opgeslagen) en meer.

Bekijk ``.platform.app.yaml``, er is al een schrijfbare *mount* voor de ``var/`` map. De ``var/`` map is de enige map waar Symfony schrijft (caches, logs, ....).

We maken een nieuwe mount voor geüploade foto's:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -41,6 +41,7 @@ applications:
             mounts:
                 "/var/cache": { source: instance, source_path: var/cache }
                 "/var/share": { source: storage, source_path: var/share }
    +            "/public/uploads": { source: storage, source_path: uploads }


             relationships:

Je kan nu de code deployen. De foto's zullen opgeslagen worden in de ``public/uploads/`` map zoals in onze lokale versie.

.. sidebar:: Verder gaan

    * `SymfonyCasts Forms tutorial`_;

    * Hoe `de weergave van Symfony formulieren in HTML aan te passen`_;

    * `Valideren van Symfony Formulieren`_;

    * De `Symfony Form Types referentie`_;

    * De `FlysystemBundle documentatie`_ , die integratie voorziet met meerdere cloud storage providers, zoals AWS S3, Azure en Google Cloud Storage;

    * De `Symfony Configuratie Parameters`_.

    * De `Symfony Validation Constraints`_;

    * De `Symfony Form Cheat Sheet`_.

.. _`CSRF-aanvallen`: https://owasp.org/www-community/attacks/csrf
.. _`form type`: https://symfony.com/doc/current/forms.html#form-types
.. _`SymfonyCasts Forms tutorial`: https://symfonycasts.com/screencast/symfony-forms
.. _`de weergave van Symfony formulieren in HTML aan te passen`: https://symfony.com/doc/current/form/form_customization.html
.. _`Valideren van Symfony Formulieren`: https://symfony.com/doc/current/forms.html#validating-forms
.. _`Symfony Form Types referentie`: https://symfony.com/doc/current/reference/forms/types.html
.. _`FlysystemBundle documentatie`: https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md
.. _`Symfony Configuratie Parameters`: https://symfony.com/doc/current/configuration.html#configuration-parameters
.. _`Symfony Validation Constraints`: https://symfony.com/doc/current/validation.html#basic-constraints
.. _`Symfony Form Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf
