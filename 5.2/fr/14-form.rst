Accepter des commentaires avec les formulaires
==============================================

.. index::
    single: Components;Form
    single: Form

Time to let our attendees give feedback on conferences. They will contribute
their comments through an *HTML form*.

Générer un *form type*
------------------------

.. index::
    single: Command;make:form

Utilisez le *Maker Bundle* pour générer une classe de formulaire :

.. code-block:: bash

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

The ``App\Form\CommentFormType`` class defines a form for the
``App\Entity\Comment`` entity:

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

A `form type`_ describes the *form fields* bound to a model. It does the data
conversion between submitted data and the model class properties. By default,
Symfony uses metadata from the ``Comment`` entity - such as the Doctrine
metadata - to guess configuration about each field. For example, the ``text``
field renders as a ``textarea`` because it uses a larger column in the
database.

Afficher un formulaire
----------------------

To display the form to the user, create the form in the controller and pass it
to the template:

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

You should never instantiate the form type directly. Instead, use the
``createForm()`` method. This method is part of ``AbstractController`` and
eases the creation of forms.

.. index::
    single: Twig;form

When passing a form to a template, use ``createView()`` to convert the data to
a format suitable for templates.

L'affichage du formulaire dans le template peut se faire via la fonction Twig ``form`` :

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

When refreshing a conference page in the browser, note that each form field
shows the right HTML widget (the data type is derived from the model):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

The ``form()`` function generates the HTML form based on all the information
defined in the Form type. It also adds ``enctype=multipart/form-data`` on the
``<form>`` tag as required by the file upload input field. Moreover, it takes
care of displaying error messages when the submission has some errors.
Everything can be customized by overriding the default templates, but we won't
need it for this project.

Personnaliser un form type
--------------------------

Even if form fields are configured based on their model counterpart, you can
customize the default configuration in the form type class directly:

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

Note that we have added a submit button (that allows us to keep using the
simple ``{{ form(comment_form) }}`` expression in the template).

Some fields cannot be auto-configured, like the ``photoFilename`` one. The
``Comment`` entity only needs to save the photo filename, but the form has to
deal with the file upload itself. To handle this case, we have added a field
called ``photo`` as un-``mapped`` field: it won't be mapped to any property on
``Comment``. We will manage it manually to implement some specific logic (like
storing the uploaded photo on the disk).

As an example of customization, we have also modified the default label for
some fields.

The image constraint works by checking the mime type; require the Mime
component to make it work:

.. code-block:: bash

    $ symfony composer req mime

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Valider des modèles
--------------------

The Form Type configures the frontend rendering of the form (via some HTML5
validation). Here is the generated HTML form:

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

The form uses the ``email`` input for the comment email and makes most of the
fields ``required``. Note that the form also contains a ``_token`` hidden field
to protect the form from `CSRF attacks
<https://owasp.org/www-community/attacks/csrf>`_.

But if the form submission bypasses the HTML validation (by using an HTTP
client that does not enforce these validation rules like cURL), invalid data
can hit the server.

Nous devons également ajouter certaines contraintes de validation à l'entité ``Comment`` :

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

Gérer un formulaire
--------------------

Le code que nous avons écrit jusqu'à présent est suffisant pour afficher le formulaire.

We should now handle the form submission and the persistence of its information
to the database in the controller:

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

When the form is submitted, the ``Comment`` object is updated according to the
submitted data.

The conference is forced to be the same as the one from the URL (we removed it
from the form).

If the form is not valid, we display the page, but the form will now contain
submitted values and error messages so that they can be displayed back to the
user.

Try the form. It should work well and the data should be stored in the
database (check it in the admin backend). There is one problem though: photos.
They do not work as we have not handled them yet in the controller.

Uploader des fichiers
---------------------

Uploaded photos should be stored on the local disk, somewhere accessible by the
frontend so that we can display them on the conference page. We will store them
under the ``public/uploads/photos`` directory:

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

To manage photo uploads, we create a random name for the file. Then, we move
the uploaded file to its final location (the photo directory). Finally, we
store the filename in the Comment object.

.. index::
    single: Container;Bind
    single: Bind

Notice the new argument on the ``show()`` method? ``$photoDir`` is a string
and not a service. How can Symfony know what to inject here? The Symfony
Container is able to store *parameters* in addition to services. Parameters are
scalars that help configure services. These parameters can be injected into
services explicitly, or they can be *bound by name*:

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

The ``bind`` setting allows Symfony to inject the value whenever a service has
a ``$photoDir`` argument.

Try to upload a PDF file instead of a photo. You should see the error messages
in action. The design is quite ugly at the moment, but don't worry, everything
will turn beautiful in a few steps when we will work on the design of the
website. For the forms, we will change one line of configuration to style all
form elements.

Déboguer des formulaires
-------------------------

When a form is submitted and something does not work quite well, use the "Form"
panel of the Symfony Profiler. It gives you information about the
form, all its options, the submitted data and how they are converted
internally. If the form contains any errors, they will be listed as well.

Le workflow classique d'un formulaire est le suivant :

* The form is displayed on a page;

* The user submits the form via a POST request;

* The server redirects the user to another page or the same page.

But how can you access the profiler for a successful submit request? Because
the page is immediately redirected, we never see the web debug toolbar for the
POST request. No problem: on the redirected page, hover over the left "200"
green part. You should see the "302" redirection with a link to the profile (in
parenthesis).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Cliquez dessus pour accéder au profileur de la requête POST, et allez dans le panneau "Form" :

.. code-block:: bash
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Afficher les photos uploadées dans l'interface d'administration
----------------------------------------------------------------

The admin backend is currently displaying the photo filename, but we want
to see the actual photo:

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

Exclure les photos uploadées de Git
------------------------------------

Don't commit yet! We don't want to store uploaded images in the Git repository.
Add the ``/public/uploads`` directory to the ``.gitignore`` file:

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Stocker les fichiers uploadés sur les serveurs de production
-------------------------------------------------------------

The last step is to store the uploaded files on production servers. Why would
we have to do something special? Because most modern cloud platforms use
read-only containers for various reasons. SymfonyCloud is no exception.

Not everything is read-only in a Symfony project. We try hard to generate as
much cache as possible when building the container (during the cache warmup
phase), but Symfony still needs to be able to write somewhere for the user
cache, the logs, the sessions if they are stored on the filesystem, and more.

Have a look at ``.symfony.cloud.yaml``, there is already a writeable *mount*
for the ``var/`` directory. The ``var/`` directory is the only directory where
Symfony writes (caches, logs, ...).

Créez un nouveau montage pour les photos uploadées :

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

You can now deploy the code and photos will be stored in the
``public/uploads/`` directory like our local version.

.. sidebar:: Going Further

    * `SymfonyCasts Forms tutorial <https://symfonycasts.com/screencast/symfony-forms>`_;

    * How to `customize Symfony Form rendering in HTML
      <https://symfony.com/doc/current/form/form_customization.html>`_;

    * `Validating Symfony Forms
      <https://symfony.com/doc/current/forms.html#validating-forms>`_;

    * The `Symfony Form Types reference
      <https://symfony.com/doc/current/reference/forms/types.html>`_;

    * The `FlysystemBundle docs
      <https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md>`_,
      which provides integration with multiple cloud storage providers, such as
      AWS S3, Azure and Google Cloud Storage;

    * The `Symfony Configuration Parameters
      <https://symfony.com/doc/current/configuration.html#configuration-parameters>`_.

    * The `Symfony Validation Constraints
      <https://symfony.com/doc/current/validation.html#basic-constraints>`_;

    * The `Symfony Form Cheat Sheet
      <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf>`_.

.. _`form type`: https://symfony.com/doc/current/forms.html#form-types
