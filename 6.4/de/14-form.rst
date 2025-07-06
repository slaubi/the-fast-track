Feedback mit Formularen annehmen
================================

.. index::
    single: Components;Form
    single: Form

Es ist an der Zeit, dass unsere Teilnehmer*innen Feedback zu Konferenzen geben. Sie werden ihre Kommentare über ein *HTML-Formular* einbringen.

Einen Form-Type generieren
--------------------------

.. index::
    single: Command;make:form

Verwende das Maker-Bundle, um eine Formularklasse zu generieren:

.. code-block:: terminal

    $ symfony console make:form CommentType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

Die ``App\Form\CommentType``-Klasse definiert ein Formular für die ``App\Entity\Comment``-Entity:

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
        public function buildForm(FormBuilderInterface $builder, array $options)
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

        public function configureOptions(OptionsResolver $resolver)
        {
            $resolver->setDefaults([
                'data_class' => Comment::class,
            ]);
        }
    }

Ein `Form-Type`_ beschreibt die mit einem Modell verknüpften *Formularfelder*. Er übernimmt die Datenkonvertierung zwischen den übermittelten Daten und den Properties/Eigenschaften der Modellklasse. Standardmäßig verwendet Symfony Metadaten aus der ``Comment``-Entity – wie z. B. die Doctrine Metadaten – um die Konfiguration für jedes Feld zu erraten. Beispielsweise wird das ``text``-Feld als ``textarea`` dargestellt, weil es eine größere Spalte in der Datenbank verwendet.

Formulare anzeigen
------------------

Um den Benutzer*innen das Formular anzuzeigen, erstellst Du das Formular im Controller und übergibst es an das Template:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 19,29

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,7 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Comment;
     use App\Entity\Conference;
    +use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    @@ -23,6 +25,9 @@ final class ConferenceController extends AbstractController
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $comment = new Comment();
    +        $form = $this->createForm(CommentType::class, $comment);
    +
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    @@ -31,6 +36,7 @@ final class ConferenceController extends AbstractController
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    +            'comment_form' => $form,
             ]);
         }
     }

Du solltest den Form-Type niemals direkt instanziieren. Verwende stattdessen die ``createForm()``-Methode. Diese Methode ist Teil vom ``AbstractController`` und erleichtert die Erstellung von Formularen.

.. index::
    single: Twig;form

Die Darstellung des Formulars im Template kann über die Twig-Funktion ``form`` erfolgen:

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

Nach dem Aktualisieren einer Konferenzseite im Browser siehst Du, dass jedes Formularfeld das richtige HTML-Widget anzeigt (der Datentyp wird aus dem Modell abgeleitet):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Die ``form()``-Funktion generiert das HTML-Formular auf der Grundlage aller im Form-Type definierten Informationen. Es ergänzt auch den ``<form>``-Tag um ``enctype=multipart/form-data``, weil das Eingabefeld für den Datei-Upload dies erfordert. Außerdem kümmert es sich um die Anzeige von Fehlermeldungen, falls die Eingabe fehlerhaft ist. Alles kann durch Überschreiben der Standard-Templates angepasst werden, aber wir werden das für dieses Projekt nicht tun müssen.

Einen Form-Type anpassen
------------------------

Auch wenn Formularfelder basierend auf ihrem Modellgegenstück konfiguriert werden, kannst Du die Standardkonfiguration in der Form-Type-Klasse direkt anpassen:

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

Beachte, dass wir einen Submit-Button hinzugefügt haben (der es uns ermöglicht, den einfachen ``{{ form(comment_form) }}`` Ausdruck in der Vorlage weiterhin zu verwenden).

Einige Felder können nicht automatisch konfiguriert werden, etwa ``photoFilename``. Die ``Comment``-Entity muss nur den Dateinamen des Fotos speichern, aber das Formular muss sich mit dem Hochladen der Datei selbst befassen. Um diesen Fall zu behandeln, haben wir ein Property namens ``photo`` als un-``mapped`` Feld hinzugefügt: es gehört zu keinem Datenbank-Feld der ``Comment``-Entity. Wir werden es manuell verwalten, um eine bestimmte Logik zu implementieren (wie das Speichern des hochgeladenen Fotos auf der Festplatte).

Als Beispiel für eine Anpassung haben wir auch die Standardbezeichnung für einige Felder geändert.

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Modelle validieren
------------------

Der Form-Type konfiguriert das Frontend-Rendering des Formulars (mit etwas HTML5-Validierung). Hier ist das generierte HTML-Formular:

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

Das Formular verwendet das ``email``-Element für die E-Mail-Adresse des Autors und markiert die meisten der Felder mit ``required``. Beachte, dass das Formular auch ein verstecktes ``_token``-Feld enthält, um das Formular vor `CSRF-Angriffen`_ zu schützen.

Wenn die Formularübermittlung jedoch die HTML-Validierung umgeht (mit einem HTTP-Client, der diese Validierungsregeln nicht durchsetzt, wie cURL), können ungültige Daten auf den Server gelangen.

Wir müssen auch einige Validierungsregeln für das ``Comment``-Datenmodell hinzufügen:

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

Ein Formular verarbeiten
------------------------

Der Code, den wir bisher geschrieben haben, reicht aus, um das Formular anzuzeigen.

Wir sollten nun im Controller die Übermittlung des Formulars verarbeiten und die Informationen in der Datenbank speichern:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
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
    @@ -27,6 +33,15 @@ final class ConferenceController extends AbstractController
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

             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

Beim Absenden des Formulars wird das ``Comment``-Objekt entsprechend der übermittelten Daten aktualisiert.

Wir erzwingen, dass die Konferenz die gleiche ist, wie die aus der URL (wir haben sie aus dem Formular entfernt).

Wenn die Formulareingabe nicht gültig ist, zeigen wir die Seite an, aber das Formular enthält nun die übermittelten Werte sowie Fehlermeldungen, so dass sie dem*r Benutzer*in wieder angezeigt werden können.

Probiere das Formular aus. Es sollte gut funktionieren und die Daten sollten in der Datenbank gespeichert sein (überprüfe dies im Admin-Backend). Es gibt jedoch ein Problem: Fotos. Sie funktionieren nicht, da wir sie noch nicht im Controller behandelt haben.

Dateien hochladen
-----------------

Hochgeladene Fotos sollten auf der lokalen Festplatte gespeichert werden, an einem Ort, der über das Frontend zugänglich ist, damit wir sie auf der Konferenzseite anzeigen können. Wir werden sie unter dem ``public/uploads/photos``-Verzeichnis speichern.

.. index::
    single: Attribute;Autowire
    single: Autowire

Da wir keine festen (hardcoded) Verzeichnis-Pfad im Code haben wollen, brauchen wir eine Möglichkeit ihn global in der Konfiguration zu speichern. Der Symfony-Container kann zusätzlich zu Diensten, auch *Parameter* speichern, welche Skalare sind, die helfen Dienste zu konfigurieren:

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

Wir haben bereits gesehen wie Dienste automatisch in Constructor-Argumente injiziert werden. Container-Parameter können wir direkt via ``Autowire``-Attribute injizieren.

Jetzt wissen wir, was wir alles brauchen, um die Foto-Upload-Logik zu implementieren, die die hochgeladene Datei an ihren endgültigen Speicherort speichert:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,6 +9,7 @@ use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    @@ -29,13 +30,22 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    -    {
    +    public function show(
    +        Request $request,
    +        Conference $conference,
    +        CommentRepository $commentRepository,
    +        #[Autowire('%photo_dir%')] string $photoDir,
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

Um Foto-Uploads zu verwalten, erstellen wir einen zufälligen Namen für die Datei. Dann verschieben wir die hochgeladene Datei an ihren endgültigen Speicherort (das Fotoverzeichnis). Schließlich speichern wir den Dateinamen im Comment-Objekt.

Versuche, eine PDF-Datei anstelle eines Fotos hochzuladen. Du solltest die Fehlermeldungen in Aktion sehen. Das Design ist im Moment ziemlich hässlich, aber keine Sorge, in ein paar Schritten wird alles schön, wenn wir am Design der Website arbeiten. Für die Formulare werden wir eine Zeile der Konfiguration ändern, um alle Formularelemente zu verschönern.

Formulare debuggen
------------------

Wenn ein Formular abgeschickt wird und etwas nicht klappt, verwende das "Formular"-Panel des Symfony Profilers. Es gibt Dir Informationen über das Formular, seine Optionen, die übermittelten Daten und wie sie intern konvertiert werden. Falls das Formular Fehler enthält, werden diese ebenfalls angezeigt.

Der typische Formular-Workflow sieht so aus:

* Das Formular wird auf einer Seite angezeigt;

* Der*ie Benutzer*in sendet das Formular über eine POST-Anfrage;

* Der Server leitet den*ie Benutzer*in auf eine andere oder die gleiche Seite weiter.

Aber wie kannst Du auf den Profiler für eine erfolgreiche Anfrage zugreifen? Da die Seite sofort umgeleitet wird, sehen wir nie die Web-Debug-Toolbar für die POST-Anfrage. Kein Problem: Fahre auf der umgeleiteten Seite mit der Maus über den linken grünen Teil mit der "200". Du solltest die "302" Umleitung mit einem Link zum Profil sehen (in Klammern).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Klicke darauf, um auf das POST-Request-Profil zuzugreifen, und gehe zum "Forms"-Panel:

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Hochgeladene Fotos im Admin-Backend anzeigen
--------------------------------------------

Das Admin-Backend zeigt derzeit den Dateinamen des Fotos an, aber wir wollen das aktuelle Foto sehen:

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

Hochgeladene Fotos von Git ausschließen
----------------------------------------

Noch nicht committen! Wir wollen keine hochgeladenen Bilder im Git-Repository speichern. Füge das Verzeichnis ``/public/uploads`` zur ``.gitignore``-Datei hinzu:

.. code-block:: diff
    :caption: patch_file

    --- i/.gitignore
    +++ w/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Hochgeladene Dateien auf Produktivservern speichern
---------------------------------------------------

Der letzte Schritt besteht darin, die hochgeladenen Dateien auf Produktionsservern zu speichern. Warum sollten wir etwas Besonderes tun müssen? Weil die meisten modernen Cloud-Plattformen, aus verschiedenen Gründen, schreibgeschützte Container verwenden. Platform.sh bildet dabei keine Ausnahme.

In einem Symfony-Projekt ist nicht alles schreibgeschützt. Wir versuchen, beim Erstellen des Containers (während der Aufwärmphase des Caches) so viel Cache wie möglich zu erzeugen, aber Symfony muss immer noch in der Lage sein, irgendwo schreiben zu können – etwa den Cache für den*ie Benutzer*in, Logs, die Sessions (wenn sie im Dateisystem gespeichert werden) uvm.

Wirf einen Blick in ``.platform.app.yaml``, da gibt es bereits einen beschreibbaren *mount* für das Verzeichnis ``var/``. Es ist das einzige Verzeichnis, in das Symfony schreibt (Caches, Logs, ...).

Lass uns einen neuen Mount für hochgeladene Fotos erstellen:

.. code-block:: diff
    :caption: patch_file

    --- i/.platform.app.yaml
    +++ w/.platform.app.yaml
    @@ -31,6 +31,7 @@ web:

     mounts:
         "/var/cache": { source: local, source_path: var/cache }
    +    "/public/uploads": { source: local, source_path: uploads }
         

     relationships:

Du kannst den Code jetzt deployen und Fotos werden wie unsere lokale Version im ``public/uploads/``-Verzeichnis gespeichert.

.. sidebar:: Weiterführendes

    * `SymfonyCasts Tutorial für Formulare`_;

    * Wie man die `Darstellung von Symfony-Formularen in HTML anpasst`_;

    * `Validierung von Symfony-Formularen`_;

    * Die `Referenz der Symfony-Form-Typen`_;

    * Die `FlysystemBundle Dokumentation`_, welche die Integration mit mehreren Cloud-Speicheranbietern wie AWS S3, Azure und Google Cloud Storage ermöglicht;

    * Die `Symfony-Konfigurationsparameter`_.

    * Die `Symfony Validierungsregeln`_;

    * Das `Symfony Form Cheat Sheet`_.

.. _`CSRF-Angriffen`: https://owasp.org/www-community/attacks/csrf
.. _`Form-Type`: https://symfony.com/doc/current/forms.html#form-types
.. _`SymfonyCasts Tutorial für Formulare`: https://symfonycasts.com/screencast/symfony-forms
.. _`Darstellung von Symfony-Formularen in HTML anpasst`: https://symfony.com/doc/current/form/form_customization.html
.. _`Validierung von Symfony-Formularen`: https://symfony.com/doc/current/forms.html#validating-forms
.. _`Referenz der Symfony-Form-Typen`: https://symfony.com/doc/current/reference/forms/types.html
.. _`FlysystemBundle Dokumentation`: https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md
.. _`Symfony-Konfigurationsparameter`: https://symfony.com/doc/current/configuration.html#configuration-parameters
.. _`Symfony Validierungsregeln`: https://symfony.com/doc/current/validation.html#basic-constraints
.. _`Symfony Form Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf
