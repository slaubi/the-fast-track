Formulare pentru recenzii
=========================

.. index::
    single: Components;Form
    single: Form

E timpul să-i lăsăm pe participanții noștri să-și dea părerea cu privire la conferințe. Vor contribui cu comentarii printr-un *formular HTML*.

Generarea unui tip de formular
------------------------------

.. index::
    single: Command;make:form

Folosește pachetul Maker pentru a genera o clasă de formular:

.. code-block:: bash

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

Clasa ``App\Form\CommentFormType`` definește un formular pentru entitatea ``App\Entity\Comment``:

.. code-block:: php
    :caption: src/App/Form/CommentFormType.php
    :class: ignore

    namespace App\Form;

    use App\Entity\Comment;
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\Component\OptionsResolver\OptionsResolver;

    class CommentFormType extends AbstractType
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

Un `tip de formular`_ descrie *câmpurile formularului* legat de un model. Face conversia datelor între datele transmise și proprietățile clasei de model. În mod implicit, Symfony folosește metadate de la entitatea ``Comentariu``, cum ar fi metadatele Doctrine, pentru a ghici configurația fiecărui câmp. De exemplu, câmpul ``text`` se redă ca ``textarea`` deoarece folosește o coloană mai mare în baza de date.

Afișarea unui formular
-----------------------

Pentru a afișa formularul utilizatorilor, creează formularul în controler și transmite-l șablonului:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18,24

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,7 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Comment;
     use App\Entity\Conference;
    +use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    @@ -31,6 +33,9 @@ class ConferenceController extends AbstractController
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $comment = new Comment();
    +        $form = $this->createForm(CommentFormType::class, $comment);
    +
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    @@ -39,6 +44,7 @@ class ConferenceController extends AbstractController
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::PAGINATOR_PER_PAGE),
    +            'comment_form' => $form->createView(),
             ]));
         }
     }

Niciodată nu ar trebui să inițiezi direct tipul formularului. În schimb, utilizează metoda ``createForm()``. Această metodă face parte din ``AbstractController`` și ușurează crearea formularelor.

.. index::
    single: Twig;form

Când transmiți un formular la un șablon, utilizează ``createView()`` pentru a converti datele într-un format potrivit pentru șabloane.

Afișarea formularului în șablon se poate face prin funcția Twig ``form``:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 10

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -30,4 +30,8 @@
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}
    +
    +    <h2>Add your own feedback</h2>
    +
    +    {{ form(comment_form) }}
     {% endblock %}

Când actualizezi o pagină de conferință în browser, reține că fiecare câmp de formular arată widgetul HTML corect (tipul de date este derivat din model):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Funcția ``form()`` generează formularul HTML pe baza tuturor informațiilor definite în tipul Form. De asemenea, adaugă ``enctype=multipart/form-data`` pe eticheta ``<form>``, așa cum este necesar pentru câmpul de încărcare a fișierului. Mai mult, are grijă să afișeze mesaje de eroare atunci când trimiterea are unele erori. Totul poate fi personalizat prin suprascrierea șabloanelor implicite, dar nu vom avea nevoie de el pentru acest proiect.

Personalizarea unui tip de formular
-----------------------------------

Chiar dacă câmpurile de formular sunt configurate pe baza omologului lor de model, poți personaliza direct configurația implicită din clasa formularului:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Form/CommentFormType.php
    +++ b/src/Form/CommentFormType.php
    @@ -4,20 +4,31 @@ namespace App\Form;

     use App\Entity\Comment;
     use Symfony\Component\Form\AbstractType;
    +use Symfony\Component\Form\Extension\Core\Type\EmailType;
    +use Symfony\Component\Form\Extension\Core\Type\FileType;
    +use Symfony\Component\Form\Extension\Core\Type\SubmitType;
     use Symfony\Component\Form\FormBuilderInterface;
     use Symfony\Component\OptionsResolver\OptionsResolver;
    +use Symfony\Component\Validator\Constraints\Image;

     class CommentFormType extends AbstractType
     {
         public function buildForm(FormBuilderInterface $builder, array $options)
         {
             $builder
    -            ->add('author')
    +            ->add('author', null, [
    +                'label' => 'Your name',
    +            ])
                 ->add('text')
    -            ->add('email')
    -            ->add('createdAt')
    -            ->add('photoFilename')
    -            ->add('conference')
    +            ->add('email', EmailType::class)
    +            ->add('photo', FileType::class, [
    +                'required' => false,
    +                'mapped' => false,
    +                'constraints' => [
    +                    new Image(['maxSize' => '1024k'])
    +                ],
    +            ])
    +            ->add('submit', SubmitType::class)
             ;
         }

Rețineți că am adăugat un buton de expediere (care ne permite să continuăm folosind expresia simplă ``{{ form(comment_form) }}`` din șablon).

Unele câmpuri nu pot fi configurate automat, cum ar fi ``photoFilename``. Entitatea ``Comment`` trebuie doar să salveze numele de fișier foto, dar formularul trebuie să se ocupe de încărcarea fișierului. Pentru a rezolva acest aspect, am adăugat un câmp numit ``photo`` drept câmp non-``mapped``: acesta nu va fi mapat în nicio proprietate din ``Comment``. Îl vom gestiona manual pentru a implementa o logică specifică (cum ar fi stocarea fotografiei încărcate pe disc).

Ca exemplu de personalizare am modificat și eticheta implicită pentru unele câmpuri.

Constrângerea imaginii funcționează prin verificarea tipului MIME; necesită componenta Mime pentru a funcționa:

.. code-block:: bash

    $ symfony composer req mime

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Validarea modelelor
-------------------

Form Type configurează redarea formularului pe frontend (prin unele validări HTML5). Iată formularul HTML generat:

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

Formularul utilizează câmpul ``email`` pentru e-mailul de comentariu și face cele mai multe câmpuri ``required``. Reține că formularul conține și un câmp ascuns `` _token`` pentru a proteja formularul de `atacurile CSRF <https://www.owasp.org/index.php/Cross-Site_Request_Forgery_ (CSRF)>`_.

Dar dacă trimiterea formularului ocolește validarea HTML (prin utilizarea unui client HTTP care nu aplică aceste reguli de validare, cum ar fi cURL), datele nevalide pot ajunge la server.

De asemenea, trebuie să adăugăm câteva restricții de validare la modelul de date ``Comment``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -4,6 +4,7 @@ namespace App\Entity;

     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Validator\Constraints as Assert;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    @@ -21,16 +22,20 @@ class Comment
         /**
          * @ORM\Column(type="string", length=255)
          */
    +    #[Assert\NotBlank]
         private $author;

         /**
          * @ORM\Column(type="text")
          */
    +    #[Assert\NotBlank]
         private $text;

         /**
          * @ORM\Column(type="string", length=255)
          */
    +    #[Assert\NotBlank]
    +    #[Assert\Email]
         private $email;

         /**

Gestionarea unui formular
-------------------------

Codul pe care l-am scris până acum este suficient pentru a afișa formularul.

Acum ar trebui să ne ocupăm de expedierea formularului și persistența informațiilor sale în baza de date din controler:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    @@ -16,10 +17,12 @@ use Twig\Environment;
     class ConferenceController extends AbstractController
     {
         private $twig;
    +    private $entityManager;

    -    public function __construct(Environment $twig)
    +    public function __construct(Environment $twig, EntityManagerInterface $entityManager)
         {
             $this->twig = $twig;
    +        $this->entityManager = $entityManager;
         }

         #[Route('/', name: 'homepage')]
    @@ -35,6 +38,15 @@ class ConferenceController extends AbstractController
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
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

Când formularul este expediat, obiectul ``Comment`` este actualizat în conformitate cu datele transmise.

Conferința este obligată să fie aceeași cu cea de pe URL (am eliminat-o din formular).

Dacă formularul nu este valid, afișăm pagina, dar formularul va conține acum valorile trimise și mesajele de eroare, astfel încât acestea să poată fi afișate înapoi utilizatorului.

Încearcă formularul. Ar trebui să funcționeze bine, iar datele ar trebui să fie stocate în baza de date (verifică-le în backend-ul admin). Există însă o problemă: fotografii. Ele nu funcționează deoarece nu le-am gestionat încă în controler.

Încărcarea fișierelor
------------------------

Fotografiile încărcate trebuie stocate pe discul local, undeva accesibil de frontend, astfel încât să le putem afișa pe pagina de conferință. Le vom stoca în directorul ``public/upload/photos``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,6 +9,7 @@ use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
    @@ -34,13 +35,22 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
             $form->handleRequest($request);
             if ($form->isSubmitted() && $form->isValid()) {
                 $comment->setConference($conference);
    +            if ($photo = $form['photo']->getData()) {
    +                $filename = bin2hex(random_bytes(6)).'.'.$photo->guessExtension();
    +                try {
    +                    $photo->move($photoDir, $filename);
    +                } catch (FileException $e) {
    +                    // unable to upload the photo, give up
    +                }
    +                $comment->setPhotoFilename($filename);
    +            }

                 $this->entityManager->persist($comment);
                 $this->entityManager->flush();

Pentru a gestiona încărcările de fotografii, creăm un nume aleatoriu pentru fișier. Apoi, mutăm fișierul încărcat în locația sa finală (directorul foto). În cele din urmă, stocăm numele fișierului în obiectul Comment.

.. index::
    single: Container;Bind
    single: Bind

Ai observat noul argument a metodei ``show()``? ``$photoDir`` este un șir și nu un serviciu. Cum poate Symfony să știe ce să injecteze aici? Symfony Container este capabil să stocheze *parametrii * pe lângă servicii. Parametrii sunt scalari care ajută la configurarea serviciilor. Acești parametri pot fi injectați în mod explicit în servicii sau pot fi *legați prin nume*:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -10,6 +10,8 @@ services:
         _defaults:
             autowire: true      # Automatically injects dependencies in your services.
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
    +        bind:
    +            $photoDir: "%kernel.project_dir%/public/uploads/photos"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

Setarea ``bind`` permite lui Symfony să injecteze valoarea ori de câte ori un serviciu are un argument `` $photoDir``.

Încearcă să încărci un fișier PDF în loc de fotografie. Ar trebui să vezi mesajele de eroare în acțiune. Designul este destul de urât în acest moment, dar nu îți fă griji, totul va deveni frumos în câțiva pași când vom lucra la designul site-ului. Pentru formulare, vom schimba o linie de configurare în stilul tuturor elementelor de formular.

Depanarea formularelor
----------------------

Când un formular este expediat și ceva nu funcționează destul de bine, utilizează opțiunea „Form” al depanatorului Symfony. Îți oferă informații despre formular, toate opțiunile sale, datele transmise și modul în care acestea sunt convertite în interior. Dacă formularul conține erori, acestea vor fi afișate și ele.

Fluxul de lucru tipic este urmatorul:

* Formularul este afișat pe o pagină;

* Utilizatorul trimite formularul printr-o solicitare POST;

* Serverul redirecționează utilizatorul către o altă pagină sau aceeași pagină.

Dar cum poți accesa depanatorul pentru a expedia o cerere cu succes? Deoarece pagina este redirecționată imediat, nu vom vedea niciodată bara de instrumente de depanare web pentru solicitarea POST. Nicio problemă: pe pagina redirecționată, treci peste partea verde "200" din stânga. Ar trebui să vezi redirecționarea "302" cu un link către profil (în paranteză).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Fă clic pe el pentru a accesa profilul solicitării POST și accesează panoul „Form”:

.. code-block:: bash
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Afișarea fotografiilor încărcate în Backendul Admin
-------------------------------------------------------

Backend-ul de administrare afișează în prezent numele fișierului foto, dar vrem să vedem fotografia reală:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
    @@ -9,6 +9,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
     use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\ImageField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
     use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;
    @@ -45,7 +46,9 @@ class CommentCrudController extends AbstractCrudController
             yield TextareaField::new('text')
                 ->hideOnIndex()
             ;
    -        yield TextField::new('photoFilename')
    +        yield ImageField::new('photoFilename')
    +            ->setBasePath('/uploads/photos')
    +            ->setLabel('Photo')
                 ->onlyOnIndex()
             ;

Excluzând fotografiile încărcate din Git
-------------------------------------------

Nu salva încă! Nu dorim să stocăm imagini încărcate în repozitoriul Git. Adăugă directorul ``/public/uploads`` în fișierul ``.gitignore``:

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Stocarea fișierelor încărcate pe serverele de producție
-----------------------------------------------------------

Ultimul pas este stocarea fișierelor încărcate pe serverele de producție. De ce ar trebui să facem ceva special? Deoarece majoritatea platformelor cloud moderne folosesc containere doar în regim de citire din diverse motive. SymfonyCloud nu face excepție.

Nu totul are permisiuni de citire într-un proiect Symfony. Încercăm din răsputeri să generăm cât mai multă memorie cache atunci când construim containerul (în faza de încălzire a cache-ului). Dar Symfony trebuie să poată scrie undeva pentru cache-ul utilizatorului, jurnalele, sesiunile și alte elemente stocate în sistemul de fișiere.

Aruncă o privire la ``.symfony.cloud.yaml``, există deja o *montare* în regim de scriere pentru directorul ``var/``. Directorul ``var/`` este singurul director în care Symfony scrie (cache, jurnale, ...).

Să creăm un nouă cale de montare pentru fotografiile încărcate:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -36,6 +36,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    +    "/public/uploads": { source: local, source_path: uploads }

     hooks:
         build: |

Acum poți lansa codul și fotografiile vor fi stocate în directorul ``public/upload/``, similar versiunii noastre locale.

.. sidebar:: Mergând mai departe

    * `Tutorialul SymfonyCasts - Formulare <https://symfonycasts.com/screencast/symfony-forms>`_;

    * Cum să personalizezi redarea formularelor Symfony în HTML <https://symfony.com/doc/current/form/form_customization.html>`_;

    * `Validarea formularelor Symfony <https://symfony.com/doc/current/forms.html#validating-forms>`_;

    * Referința „Tipuri de formulare Symfony <https://symfony.com/doc/current/reference/forms/types.html>`_;

    * `FlysystemBundle docs <https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md>`_, care asigură integrarea cu mai mulți furnizori de stocare în cloud, cum ar fi AWS S3, Azure și Google Cloud Storage;

    * `Parametri de configurare Symfony <https://symfony.com/doc/current/configuration.html#configuration-parameters>`_.

    * `Constrângerile de validare Symfony <https://symfony.com/doc/current/validation.html#basic-constraints>`_;

    * Notițe `Formulare Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf>`_.

.. _`tip de formular`: https://symfony.com/doc/current/forms.html#form-types
