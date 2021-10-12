Przyjmowanie informacji zwrotnych za pomocą formularzy
=======================================================

.. index::
    single: Components;Form
    single: Form

Czas pozwolić naszym uczestnikom wyrazić swoją opinię na temat konferencji. Będą oni dodawać swoje komentarze poprzez *formularz HTML* .

Generowanie klasy formularza (ang. form type)
---------------------------------------------

.. index::
    single: Command;make:form

Użyj Maker Bundle, aby wygenerować klasę formularza:

.. code-block:: bash

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

Klasa ``App\Form\CommentFormType`` definiuje formularz dla encji ``App\Entity\Comment``:

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

Klasa formularza (ang. `form type`_) opisuje *pola formularza* związane z modelem. Wykonuje konwersję danych pomiędzy przesłanymi danymi a właściwościami klasy modelu. Domyślnie Symfony używa metadanych z encji ``Comment`` – takich jak metadane Doctrine – aby odgadnąć konfigurację każdego pola. Na przykład, właściwość typu ``text`` renderowane jest jako pole ``textarea`` ponieważ wykorzystuje większą kolumnę w bazie danych.

Wyświetlanie formularza
------------------------

Aby wyświetlić formularz użytkownikowi, utwórz go w kontrolerze i przekaż do szablonu:

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

Nigdy nie powinno się tworzyć instancji klasy formularza (ang. form type) bezpośrednio. Zamiast tego, wykorzystaj metodę ``createForm()``. Metoda ta jest częścią ``AbstractController`` i ułatwia tworzenie formularzy.

.. index::
    single: Twig;form

Przy przekazywaniu formularza do szablonu należy użyć metody ``createView()``, aby dokonać konwersji danych do formatu odpowiedniego dla szablonów.

W celu wyświetlenia formularza w szablonie można skorzystać z funkcji ``form`` biblioteki Twig:

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

Podczas odświeżania strony konferencji w przeglądarce zwróć uwagę, że każde pole formularza pokazuje właściwy widżet HTML (dobrany na podstawie modelu):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Funkcja ``form()`` generuje formularz HTML na podstawie wszystkich informacji zdefiniowanych w klasie formularza (ang. form type). Dodaje również ``enctype=multipart/form-data`` do elementu ``<form>`` zgodnie z wymaganiami pola wgrywania plików. Co więcej, funkcja ta zadba o wyświetlanie komunikatów o błędach, gdy żądanie zawiera błędy. Wszystko można dostosować poprzez nadpisanie domyślnych szablonów, ale nie będzie nam to potrzebne w tym projekcie.

Dostosowywanie klasy formularza (ang. form type)
------------------------------------------------

Nawet jeśli pola formularza są konfigurowane na podstawie ich odpowiednika modelu, można dostosować domyślną konfigurację bezpośrednio w klasie formularza (ang. form type):

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

Zauważ, że dodaliśmy przycisk wyślij (ang. submit) (który pozwala nam używać prostego wyrażenia ``{{ form(comment_form) }}`` w szablonie).

Niektóre pola nie mogą być automatycznie konfigurowane, tak jak to ma miejsce w przypadku pola ``photoFilename``. Encja ``Comment`` musi tylko zapisać nazwę pliku zdjęcia, ale formularz musi zająć się wgraniem pliku. Do obsługi tego przypadku dodaliśmy pole o nazwie ``photo``, które nie jest ``mapowane`` do żadnej właściwości w ``Comment`` . Będziemy zarządzać nim ręcznie, aby zaimplementować określone schematy działań (np. przechowywanie przesłanego zdjęcia na dysku).

Zmodyfikowaliśmy również domyślną etykietę dla niektórych pól, aby zademonstrować możliwości dostosowywania.

Ograniczenie Image działa dzięki sprawdzeniu typu MIME i wymaga zainstalowania komponentu Mime:

.. code-block:: bash

    $ symfony composer req mime

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Walidacja modeli
----------------

Klasa formularza (ang. form type) konfiguruje renderowanie formularza na frontendzie (poprzez niektóre walidacje HTML5). Oto wygenerowany formularz HTML:

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

Formularz wykorzystuje pole typu ``email`` do przekazania adresu e-mail komentującego i czyni większość pól wymaganymi (ang. ``required``). Zauważ też, że występuje tutaj ukryte pole ``_token``, które jest częścią mechanizmu chroniącego przed `atakami CSRF <https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)>`_.

Jeśli jednak przesłanie formularza omija walidację HTML (przy użyciu klienta HTTP, który nie egzekwuje tych zasad walidacji jak np. cURL), nieprawidłowe dane mogą trafić na serwer.

Musimy dodać również ograniczenia walidacji (ang. validation constraints) dotyczące walidacji modelu danych ``Comment``:

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

Obsługa formularza
-------------------

Napisany przez nas do tej pory kod jest wystarczający do wyświetlenia formularza.

Powinniśmy teraz zająć się przesyłaniem formularzy i zapisaniem dostarczonych przez nie danych do bazy danych w kontrolerze:

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

Po wysłaniu formularza, obiekt ``Comment`` jest aktualizowany zgodnie z przesłanymi danymi.

Konferencja musi być taka sama jak ta z adresu URL (usunęliśmy ją z formularza).

Jeśli formularz nie jest poprawny, wyświetlamy stronę, ale formularz będzie teraz zawierał przesłane wartości i komunikaty o błędach, tak aby można je było wyświetlić z powrotem użytkownikowi.

Spróbuj użyć formularza. Powinien działać dobrze, a dane powinny zostać zapisane w bazie danych (sprawdź w panelu administracyjnym). Jest jednak jeden problem: zdjęcia. Zapis zdjęć jeszcze nie działa, ponieważ nie dodaliśmy jego obsługi w kontrolerze.

Przesyłanie plików
--------------------

Przesłane zdjęcia powinny być przechowywane na dysku lokalnym, w publicznym katalogu dostępnym dla frontendu, abyśmy mogli je wyświetlić na stronie konferencji. Będziemy je przechowywać w katalogu  ``public/uploads/photos``:

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

Aby zarządzać przesyłanymi zdjęciami, tworzymy losową nazwę pliku. Następnie przenosimy przesłany plik do jego ostatecznej lokalizacji (katalogu zdjęć). Wreszcie umieszczamy nazwę pliku w obiekcie Comment.

.. index::
    single: Container;Bind
    single: Bind

Zauważyłeś/aś nowy argument w metodzie ``show()``? ``$photoDir`` jest łańcuchem znaków, a nie usługą. Skąd Symfony może wiedzieć, co powinno wstrzyknąć? Ponieważ Symfony Container, oprócz usług, jest w stanie przechowywać również *parametry*. Parametry to wartości skalarne, które pomagają w konfiguracji usług. Parametry te mogą być wstrzykiwane bezpośrednio do usług lub mogą być *wiązane nazwą* :

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

Ustawienie ``bind`` pozwala Symfony wstrzykiwać wartość za każdym razem, gdy usługa ma argument ``$photoDir``.

Spróbuj przesłać plik PDF zamiast zdjęcia. Zobacz, jak działają komunikaty o błędach. Projekt jest w tej chwili dość brzydki, ale nie martw się, wszystko wypięknieje za chwilę, kiedy będziemy pracować nad wyglądem naszej aplikacji. Zmienimy wtedy jedną linię konfiguracji, aby ostylować wszystkie elementy formularza.

Debugowanie formularzy
----------------------

Jeśli formularz został wysłany, a coś nie działa całkiem dobrze, użyj panelu „Form” w Symfony Profiler. Zawiera informacje o formularzu, wszystkich jego opcjach, przesłanych danych i sposobie ich konwersji wewnętrznej. Jeśli formularz zawiera jakieś błędy, zostaną one również wymienione.

Typowy przepływ pracy (ang. workflow) z formularzami przebiega w ten sposób:

* Formularz jest wyświetlany na stronie;

* Użytkownik przesyła formularz za pośrednictwem żądania POST;

* Serwer przekierowuje użytkownika na inną stronę lub tę samą stronę.

Ale w jaki sposób można uzyskać dostęp do profilera prawidłowego żądania? Ponieważ strona jest natychmiast przekierowywana, nigdy nie widzimy paska narzędzi do debugowania dla żądania POST. Nie ma problemu: na przekierowanej stronie należy najechać na lewą, zieloną część z napisem „200”. Zobaczysz wtedy przekierowanie „302” z linkiem do profilu (w nawiasie).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Kliknij na niego, aby uzyskać dostęp do profilu żądania POST i przejdź do panelu "Form":

.. code-block:: bash
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Wyświetlanie załadowanych zdjęć w panelu administracyjnym
-------------------------------------------------------------

Panel administracyjny wyświetla obecnie nazwę pliku zdjęcia, ale chcemy zobaczyć rzeczywiste zdjęcie:

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

Wykluczanie załadowanych zdjęć z Gita
----------------------------------------

Nie zatwierdzaj (ang. commit) jeszcze zmian! Nie chcemy przechowywać przesłanych obrazów w repozytorium Git. Dodaj katalog ``/public/uploads`` do pliku ``.gitignore``:

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Przechowywanie przesłanych plików na serwerach produkcyjnych
--------------------------------------------------------------

Ostatnim zagadnieniem jest przechowywanie przesłanych plików na serwerach produkcyjnych. Dlaczego mielibyśmy robić coś specjalnego? Ponieważ większość nowoczesnych platform chmurowych używa – z różnych powodów – kontenerów tylko do odczytu. SymfonyCloud nie jest wyjątkiem.

Nie wszystko jest w trybie tylko do odczytu w projekcie Symfony. Bardzo staramy się umieścić jak najwięcej rzeczy w pamięci podręcznej podczas budowania kontenera (podczas fazy rozgrzewania <ang. warmup>), ale Symfony nadal musi być w stanie zapisać gdzieś zrzuty pamięci podręcznej użytkownika, logi, sesje, jeśli są one przechowywane w systemie plików, i wiele więcej.

Spójrz na plik ``.symfony.cloud.yaml``. W katalogu ``var/`` jest już zasób (ang. *mount*) z prawami zapisu. Katalog ``var/`` jest jedynym katalogiem, w którym Symfony dokonuje zapisu (zrzuty pamięci podręcznej, logi, …).

Stwórzmy nowy zasób (ang. mount) dla przesłanych zdjęć:

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

Możesz teraz wdrożyć (ang. deploy) kod, a zdjęcia będą przechowywane w katalogu ``public/uploads/``, tak jak w naszej lokalnej wersji.

.. sidebar:: Idąc dalej

    * `Samouczek SymfonyCasts – formularze <https://symfonycasts.com/screencast/symfony-forms>`_;

    * Jak `dostosować renderowanie formularza Symfony w HTML <https://symfony.com/doc/current/form/form_customization.html>`_;

    * `Walidacja formularzy Symfony <https://symfony.com/doc/current/forms.html#validating-forms>`_;

    * `Klasa formularza Symfony (ang. form type) – dokument referencyjny  <https://symfony.com/doc/current/reference/forms/types.html>`_;

    * `Dokumentacja FlysystemBundle <https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md>`_, który zapewnia integrację z wieloma dostawcami przestrzeni w chmurze, takimi jak AWS S3, Azure i Google Cloud Storage;

    * `Parametry konfiguracji Symfony <https://symfony.com/doc/current/configuration.html#configuration-parameters>`_.

    * `Reguły walidacji Symfony <https://symfony.com/doc/current/validation.html#basic-constraints>`_;

    * `Ściągawka Symfony Form <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf>`_.

.. _`form type`: https://symfony.com/doc/current/forms.html#form-types
