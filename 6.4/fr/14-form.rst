Accepter des commentaires avec les formulaires
==============================================

.. index::
    single: Components;Form
    single: Form

Il est temps de permettre aux personnes présentes de donner leur avis sur les conférences. Elles feront part de leurs commentaires au moyen d'un *formulaire HTML*.

Générer un *form type*
------------------------

.. index::
    single: Command;make:form

Utilisez le *Maker Bundle* pour générer une classe de formulaire :

.. code-block:: terminal

    $ symfony console make:form CommentType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

La classe ``App\Form\CommentType`` définit un formulaire pour l'entité ``App\Entity\Comment`` :

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

Un `form type`_ décrit les *champs de formulaire* liés à un modèle. Il effectue la conversion des données entre les données soumises et les propriétés de la classe de modèle. Par défaut, Symfony utilise les métadonnées de l'entité ``Comment``, comme les métadonnées Doctrine, pour deviner la configuration de chaque champ. Par exemple, le champ ``text`` se présente sous la forme d'un ``textarea`` parce qu'il utilise une colonne plus grande dans la base de données.

Afficher un formulaire
----------------------

Pour afficher le formulaire, créez-le dans le contrôleur et transmettez-le au template :

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 19,29

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,7 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Comment;
     use App\Entity\Conference;
    +use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    @@ -23,6 +25,9 @@ class ConferenceController extends AbstractController
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $comment = new Comment();
    +        $form = $this->createForm(CommentType::class, $comment);
    +
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    @@ -31,6 +36,7 @@ class ConferenceController extends AbstractController
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    +            'comment_form' => $form,
             ]);
         }
     }

Vous ne devriez jamais instancier directement le form type. Utilisez plutôt la méthode ``createForm()``. Cette méthode fait partie d'``AbstractController`` et facilite la création de formulaires.

.. index::
    single: Twig;form

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

Lorsque vous rafraîchissez la page d'une conférence dans le navigateur, notez que chaque champ de formulaire affiche la balise HTML appropriée (le type de données est défini à partir du modèle) :

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

La fonction ``form()`` génère le formulaire HTML en fonction de toutes les informations définies dans le form type. Elle ajoute également ``enctype=multipart/form-data`` à la balise ``<form>`` comme l'exige le champ d'upload de fichier. De plus, elle se charge d'afficher les messages d'erreur lorsque la soumission comporte des erreurs. Tout peut être personnalisé en remplaçant les templates par défaut, mais nous n'en aurons pas besoin pour ce projet.

Personnaliser un form type
--------------------------

Même si les champs de formulaire sont configurés en fonction de leur modèle associé, vous pouvez personnaliser la configuration par défaut directement dans la classe de form type :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Form/CommentType.php
    +++ b/src/Form/CommentType.php
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
    -            ->add('text')
    -            ->add('email')
    -            ->add('createdAt', null, [
    -                'widget' => 'single_text',
    +            ->add('author', null, [
    +                'label' => 'Your name',
                 ])
    -            ->add('photoFilename')
    -            ->add('conference', EntityType::class, [
    -                'class' => Conference::class,
    -                'choice_label' => 'id',
    +            ->add('text')
    +            ->add('email', EmailType::class)
    +            ->add('photo', FileType::class, [
    +                'required' => false,
    +                'mapped' => false,
    +                'constraints' => [
    +                    new Image(['maxSize' => '1024k'])
    +                ],
                 ])
    -        ;
    +            ->add('submit', SubmitType::class)
    +       ;
         }

         public function configureOptions(OptionsResolver $resolver): void

Notez que nous avons ajouté un bouton submit (qui nous permet de continuer à utiliser simplement ``{{ form(comment_form) }}`` dans le template).

Certains champs ne peuvent pas être auto-configurés, comme par exemple ``photoFilename``. L'entité ``Comment`` n'a besoin d'enregistrer que le nom du fichier photo, mais le formulaire doit s'occuper de l'upload du fichier lui-même. Pour traiter ce cas, nous avons ajouté un champ appelé ``photo`` qui est un champ non ``mapped`` : il ne sera associé à aucune propriété de ``Comment``. Nous le gérerons manuellement pour implémenter une logique spécifique (comme l'upload de la photo sur le disque).

Comme exemple de personnalisation, nous avons également modifié le libellé par défaut de certains champs.

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Valider des modèles
--------------------

Le form type configure le rendu du formulaire (grâce à un peu de validation HTML5). Voici le formulaire HTML généré :

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

Le formulaire utilise le type de champ ``email`` pour l'email du commentaire et définit la plupart des champs en ``required``. Notez qu'il contient également un champ ``_token`` caché pour nous protéger des `attaques CSRF`_.

Mais si la soumission du formulaire contourne la validation HTML (en utilisant un client HTTP comme cURL, qui n'applique pas ces règles de validation), des données invalides peuvent atteindre le serveur.

Nous devons également ajouter certaines contraintes de validation à l'entité ``Comment`` :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
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

Gérer un formulaire
--------------------

Le code que nous avons écrit jusqu'à présent est suffisant pour afficher le formulaire.

Nous devrions maintenant nous occuper de la soumission du formulaire et de la persistance de ses informations dans la base de données depuis le contrôleur :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    @@ -14,6 +15,11 @@ use Symfony\Component\Routing\Attribute\Route;

     class ConferenceController extends AbstractController
     {
    +    public function __construct(
    +        private EntityManagerInterface $entityManager,
    +    ) {
    +    }
    +
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    @@ -27,6 +33,15 @@ class ConferenceController extends AbstractController
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

Lorsque le formulaire est soumis, l'objet ``Comment`` est mis à jour en fonction des données soumises.

La conférence doit être la même que celle de l'URL (nous l'avons supprimée du formulaire).

Si le formulaire n'est pas valide, nous affichons la page, mais le formulaire contiendra maintenant les valeurs soumises et les messages d'erreur afin qu'ils puissent être affichés à l'internaute.

Essayez le formulaire. Il devrait fonctionner correctement et les données devraient être stockées dans la base de données (vérifiez-les dans l'interface d'administration). Il y a cependant un problème : les photos. Elles ne fonctionnent pas puisque nous ne les avons pas encore traitées dans le contrôleur.

Uploader des fichiers
---------------------

Les photos uploadées doivent être stockées sur le disque local, à un endroit accessible par un navigateur afin que nous puissions les afficher sur la page d'une conférence. Nous les stockerons dans le dossier ``public/uploads/photos`` :

.. index::
    single: Attribute;Autowire
    single: Autowire

Comme nous ne souhaitons pas mettre le répertoire en dur dans le code, nous devons trouver un moyen de le stocker de façon globale. Le conteneur Symfony est capable de stocker des *paramètres* (parameters) en plus des services pour permettre de les configurer :

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -4,6 +4,7 @@
     # Put parameters here that don't need to change on each machine where the app is deployed
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
    +    photo_dir: "%kernel.project_dir%/public/uploads/photos"

     services:
         # default configuration for services in *this* file

Nous avons déjà vu comment les services sont automatiquement injectés dans les arguments des constructeurs. Pour les paramètres du conteneur, nous pouvons les injecter explicitement en utilisant l'attribut ``Autowire``.

Maintenant, nous avons tout ce qu'il nous faut pour implémenter la logique nécessaire au stockage du fichier soumis sous sa destination finale :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,6 +9,7 @@ use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    @@ -29,13 +30,22 @@ class ConferenceController extends AbstractController
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

Pour gérer les uploads de photos, nous créons un nom aléatoire pour le fichier. Ensuite, nous déplaçons le fichier uploadé à son emplacement final (le répertoire photo). Enfin, nous stockons le nom du fichier dans l'objet ``Comment``.

Essayez d'uploader un fichier PDF au lieu d'une photo. Vous devriez voir les messages d'erreur en action. Le design est encore assez laid, mais ne vous inquiétez pas, tout deviendra beau en quelques étapes lorsque nous travaillerons dessus. Pour les formulaires, nous allons changer une ligne de configuration pour styliser tous leurs éléments.

Déboguer des formulaires
-------------------------

Lorsqu'un formulaire est soumis et que quelque chose ne fonctionne pas correctement, utilisez le panneau "Form" du Symfony Profiler. Il vous donne des informations sur le formulaire, toutes ses options, les données soumises et comment elles sont converties en interne. Si le formulaire contient des erreurs, elles seront également répertoriées.

Le workflow classique d'un formulaire est le suivant :

* Le formulaire est affiché sur une page ;

* L'internaute soumet le formulaire via une requête POST ;

* Le serveur redirige l'internaute, soit vers une autre page, soit vers la même page.

Mais comment pouvez-vous accéder au profileur pour une requête de soumission réussie ? Étant donné que la page est immédiatement redirigée, nous ne voyons jamais la barre d'outils de débogage Web pour la requête POST. Pas de problème : sur la page redirigée, survolez la partie verte "200" à gauche. Vous devriez voir la redirection "302" avec un lien vers le profileur (entre parenthèses).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Cliquez dessus pour accéder au profileur de la requête POST, et allez dans le panneau "Form" :

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Afficher les photos uploadées dans l'interface d'administration
----------------------------------------------------------------

L'interface d'administration affiche actuellement le nom du fichier photo, mais nous voulons voir la vraie photo :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
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

Exclure les photos uploadées de Git
------------------------------------

Ne *commitez* pas encore ! Nous ne voulons pas stocker les images uploadées dans le dépôt Git. Ajoutez le dossier ``/public/uploads`` au fichier ``.gitignore`` :

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

La dernière étape consiste à stocker les fichiers uploadés sur les serveurs de production. Pourquoi devrions-nous faire quelque chose de spécial ? Parce que la plupart des plates-formes modernes de cloud utilisent des conteneurs en lecture seule pour diverses raisons. Platform.sh n'échappe pas à cette règle.

Tout n'est pas en lecture seule dans un projet Symfony. Nous essayons de générer autant de cache que possible lors de la construction du conteneur (pendant la phase de démarrage du cache), mais Symfony doit quand même être capable d'écrire quelque part pour le cache, les logs, les sessions si elles sont stockées dans le système de fichiers, etc.

Jetez un coup d'oeil au fichier ``.platform.app.yaml``, il y a déjà un *montage* accessible en écriture pour le dossier ``var/``. Le dossier ``var/`` est le seul répertoire où Symfony écrit (caches, logs, etc.).

Créez un nouveau montage pour les photos uploadées :

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -35,6 +35,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    +    "/public/uploads": { source: local, source_path: uploads }
         

     relationships:

Vous pouvez maintenant déployer le code, et les photos seront stockées dans le dossier ``public/uploads/`` comme pour notre version locale.

.. sidebar:: Aller plus loin

    * `Tutoriel SymfonyCasts sur les formulaires`_ ;

    * Comment `personnaliser le rendu des formulaires Symfony en HTML`_ ;

    * `Validation des formulaires Symfony`_ ;

    * La `référence des form types Symfony`_ ;

    * La `documentation de FlysystemBundle`_, qui permet l'intégration avec plusieurs fournisseurs de stockage dans le cloud, tels que AWS S3, Azure et Google Cloud Storage ;

    * Les `paramètres de configuration de Symfony`_.

    * Les `contraintes de validation de Symfony`_ ;

    * La `cheat sheet des formulaires Symfony`_.

.. _`attaques CSRF`: https://owasp.org/www-community/attacks/csrf
.. _`form type`: https://symfony.com/doc/current/forms.html#form-types
.. _`Tutoriel SymfonyCasts sur les formulaires`: https://symfonycasts.com/screencast/symfony-forms
.. _`personnaliser le rendu des formulaires Symfony en HTML`: https://symfony.com/doc/current/form/form_customization.html
.. _`Validation des formulaires Symfony`: https://symfony.com/doc/current/forms.html#validating-forms
.. _`référence des form types Symfony`: https://symfony.com/doc/current/reference/forms/types.html
.. _`documentation de FlysystemBundle`: https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md
.. _`paramètres de configuration de Symfony`: https://symfony.com/doc/current/configuration.html#configuration-parameters
.. _`contraintes de validation de Symfony`: https://symfony.com/doc/current/validation.html#basic-constraints
.. _`cheat sheet des formulaires Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf
