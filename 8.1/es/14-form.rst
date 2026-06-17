Obteniendo realimentación con formularios
==========================================

.. index::
    single: Components;Form
    single: Form

Es hora de dejar que nuestros asistentes den su opinión sobre las conferencias. Ellos contribuirán con sus comentarios a través de un *formulario HTML* .

Generando una clase de tipo de formulario (*Form Type*)
-------------------------------------------------------

.. index::
    single: Command;make:form

Utiliza el bundle Maker para generar una clase de formulario:

.. code-block:: terminal

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

La clase ``App\Form\CommentFormType`` define un formulario para la entidad ``App\Entity\Comment``:

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

Un `tipo de formulario`_ describe los *campos de formulario* vinculados a un modelo. Realiza la conversión de datos entre los datos enviados y las propiedades de la clase de modelo. Por defecto, Symfony utiliza metadatos de la entidad ``Comment`` - como los metadatos de Doctrine - para intuir la configuración de cada campo. Por ejemplo, el campo ``text`` se muestra como un ``textarea`` porque utiliza una columna más grande en la base de datos.

Visualizando un formulario
--------------------------

Para mostrar el formulario al usuario, crea el formulario en el controlador y pásalo a la plantilla:

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

Nunca debes instanciar el tipo de formulario directamente. En su lugar, utiliza el método ``createForm()``. Este método es parte de ``AbstractController`` y facilita la creación de formularios.

.. index::
    single: Twig;form

Al pasar un formulario a una plantilla, utiliza ``createView()`` para convertir los datos a un formato adecuado para las plantillas.

La visualización del formulario en la plantilla se puede realizar a través de la función ``form`` de Twig:

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

Al actualizar una página de conferencia en el navegador, ten en cuenta que cada campo del formulario muestra el widget HTML correcto (el tipo de datos se deriva del modelo):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

La función ``form()`` genera el formulario HTML a partir de toda la información definida en el tipo de formulario. También agrega ``enctype=multipart/form-data`` en el ``<form>`` como lo requiere el campo de entrada de carga de archivos. Además, se encarga de mostrar mensajes de error cuando el envío tiene algunos errores. Todo se puede personalizar sobreescribiendo las plantillas predeterminadas, pero no lo necesitaremos para este proyecto.

Personalizando una clase de tipo de formulario
----------------------------------------------

Aunque los campos del formulario se configuran en función de su correspondiente propiedad en el modelo, es posible personalizar directamente la configuración por defecto en la clase de tipo de formulario:

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

Ten en cuenta que hemos añadido un botón de enviar (eso nos permite seguir usando la expresión simple ``{{ form(comment_form) }}`` en la plantilla).

Algunos campos no pueden ser configurados automáticamente, como es el caso de ``photoFilename``. La entidad ``Comment`` sólo necesita guardar el nombre del archivo de la foto, pero el formulario tiene que ocuparse de la carga del archivo en sí. Para gestionar este caso, hemos añadido un campo llamado ``photo`` que no está "mapeado" (un-``mapped``): no será asociado a ninguna propiedad en ``Comment``. Lo procesaremos manualmente para implementar alguna lógica específica (como almacenar la foto enviada en el disco).

Como ejemplo de personalización, también hemos modificado la etiqueta por defecto para algunos campos.

La restricción de la imagen funciona comprobando el tipo mime; se requiere el componente Mime para que funcione:

.. code-block:: terminal

    $ symfony composer req mime

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Validación de modelos
----------------------

El tipo de formulario configura cómo se muestra en el navegador (a través de alguna validación HTML5). Aquí está el formulario HTML generado:

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

El formulario utiliza el campo ``email`` para el correo electrónico del comentario y marca la mayoría de los campos como ``required`` –obligatorios–. Ten en cuenta que el formulario también contiene un campo oculto: ``_token``, para protegerlo de `ataques CSRF <https://owasp.org/www-community/attacks/csrf>`_.

Pero si el envío del formulario pasa por alto la validación HTML (utilizando un cliente HTTP que no aplica estas reglas de validación como cURL), los datos no válidos pueden llegar al servidor.

También tenemos que añadir algunas restricciones de validación en el modelo de datos de ``Comment``:

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

Manejando un formulario
-----------------------

El código que hemos escrito hasta ahora es suficiente para mostrar el formulario.

Ahora debemos ocuparnos del envío del formulario y de la persistencia de su contenido en la base de datos desde el controlador:

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

Cuando se envía el formulario, el objeto ``Comment`` se actualiza con los datos que contiene.

Se obliga a que la conferencia sea la misma que la pasada por la URL (la hemos eliminado del formulario).

Si el formulario no es válido, se muestra la página, pero ahora el formulario contendrá los valores enviados y los correspondientes mensajes de error para que el usuario pueda verlos de nuevo.

Prueba el formulario. Debería funcionar correctamente y los datos deberían actualizarse en la base de datos (compruébalo en el panel de administración). Sin embargo, hay un problema: las fotos. No funcionan ya que no las hemos procesado todavía en el controlador.

Subiendo archivos
-----------------

Las fotos subidas deben ser almacenadas en el disco local, en un lugar accesible por el frontend (navegador) para que podamos mostrarlas en la página de la conferencia. Las guardaremos en el directorio ``public/uploads/photos``:

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

Para gestionar la carga de fotos, creamos un nombre aleatorio para el archivo. Luego, movemos el archivo cargado a su ubicación final (el directorio de fotos). Finalmente, almacenamos el nombre del archivo en el objeto Comment.

.. index::
    single: Container;Bind
    single: Bind

¿Has observado el nuevo parámetro en el método ``show()``? ``$photoDir`` es una cadena y no un servicio. ¿Cómo puede saber Symfony qué inyectar aquí? El Container de Symfony es capaz de almacenar *parámetros* además de servicios. Los parámetros son valores escalares que ayudan a configurar los servicios. Estos parámetros pueden ser inyectados en los servicios explícitamente, o pueden estar *vinculados por su nombre*:

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

La configuración ``bind`` permite a Symfony inyectar el valor cada vez que un servicio tiene un argumento ``$photoDir``.

Intenta cargar un archivo PDF en lugar de una foto. Deberías ver los mensajes de error en acción. El diseño es bastante poco atractivo en este momento, pero no te preocupes, todo se mejorará en unos pocos pasos cuando trabajemos en el diseño del sitio web. Para los formularios, cambiaremos una línea de configuración para darle estilo a todos los elementos de los formularios.

Depurando formularios
---------------------

Cuando un formulario es enviado y algo no funciona del todo bien, utiliza el panel "Form" del Profiler de Symfony. Éste proporciona información sobre el formulario, todas sus opciones, los datos enviados y cómo se convierten internamente. Si el formulario contiene errores, también se detallarán.

El típico flujo de trabajo de formularios es similar al siguiente:

* El formulario se muestra en una página;

* El usuario envía el formulario a través de una solicitud POST;

* El servidor redirige al usuario a otra página o a la misma página.

Pero, ¿cómo puedes acceder al Profiler para una solicitud de envío con éxito? Debido a que la página es redirigida inmediatamente no vemos la barra de herramientas de depuración web para la petición POST. No hay problema: en la página redirigida, pasa el ratón por encima de la sección verde con un "200" de la izquierda. Deberías ver la redirección "302" con un enlace al perfil (entre paréntesis).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Haz clic en él para acceder al perfil de la petición POST, y ve al panel "Form":

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

Visualizando las fotos cargadas en el panel de administración
--------------------------------------------------------------

El panel de administración está mostrando el nombre del archivo de la foto, pero queremos ver la foto actual:

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

Excluyendo las fotos subidas de Git
-----------------------------------

¡No hagas commit todavía! No queremos almacenar imágenes subidas en el repositorio de Git. Añade el directorio ``/public/uploads`` al archivo ``.gitignore``:

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

Almacenando archivos enviados en servidores de producción
----------------------------------------------------------

El último paso es almacenar los archivos cargados en servidores de producción. ¿Por qué tenemos que tenerlo en cuenta? Porque la mayoría de las plataformas de nube modernas utilizan contenedores de sólo lectura por varias razones. Upsun no es una excepción.

No todo es de sólo lectura en un proyecto Symfony. Nos esforzamos en incluir la mayor cantidad posible de información en la caché al construir el contenedor (durante la fase de escritura de caché), pero Symfony aún necesita poder escribir en algún lugar para la caché de usuario, los registros, las sesiones si están almacenados en el sistema de archivos, y mucho más.

Echa un vistazo a ``.symfony.cloud.yaml``, ya hay un *montaje* con permisos de escritura para el directorio ``var/``. El directorio ``var/`` es el único directorio donde Symfony escribe (cachés, registros...).

Vamos a crear un nuevo montaje para almacenar las fotos subidas:

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

Ahora se puede desplegar el código y las fotos se almacenarán en el directorio ``public/uploads/`` como en nuestra versión local.

.. sidebar:: Yendo más allá

    * `Tutorial de formularios de SymfonyCasts <https://symfonycasts.com/screencast/symfony-forms>`_ ;

    * Cómo `personalizar la presentación (renderizado) de formularios Symfony en HTML <https://symfony.com/doc/current/form/form_customization.html>`_;

    * `Validando formularios de Symfony <https://symfony.com/doc/current/forms.html#validating-forms>`_;

    * La `referencia de los tipos de formularios Symfony <https://symfony.com/doc/current/reference/forms/types.html>`_;

    * La `documentación de FlysystemBundle <https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md>`_, que proporciona integración con múltiples proveedores de almacenamiento en nube, como AWS S3, Azure y Google Cloud Storage;

    * Los `parámetros de configuración de Symfony <https://symfony.com/doc/current/configuration.html#configuration-parameters>`_ .

    * Las `Restricciones de Validación de Symfony <https://symfony.com/doc/current/validation.html#basic-constraints>`_ ;

    * La `chuleta de formularios de Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf>`_.

.. _`tipo de formulario`: https://symfony.com/doc/current/forms.html#form-types
