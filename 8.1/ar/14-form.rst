قبول الملاحظات مع النماذج
===============================================

.. index::
    single: Components;Form
    single: Form

حان الوقت للسماح للحضور بإعطاء ملاحظات علي المؤتمرات. سوف يقوموا بمشاركة تعليقاتهم عن طريق *نموذج HTML*.

إنشاء نوع النموذج (Form Type)
--------------------------------------------

.. index::
    single: Command;make:form

استخدم ال Maker bundle لإنشاء فئة نموذج:

.. code-block:: terminal

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

تحدد فئة ``App\Form\CommentFormType`` نموذجًا للكيان ``App\Entity\Comment``:

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

يصف `نوع النموذج / form type`_ *حقول النموذج* المرتبطة بالميدان Model. يقوم بتحويل البيانات بين البيانات المقدمة وخصائص فئة الميدان Model. تستخدم منظومة Symfony بشكل افتراضي البيانات الوصفية من كيان `` التعليق `` - مثل  ال Doctrine Metadata  - لتخمين الترتيب (Configuration) حول كل حقل. على سبيل المثال ، يتم عرض حقل ``النص Text`` كـ ``مساحة النص Textarea`` لأنه يستخدم عمودًا أكبر (A larger column) في قاعدة البيانات.

عرض نموذج
-----------------

لعرض النموذج للمستخدم ، قم بإنشاء النموذج في ال Controller  وتمريره إلى ال Template :

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

لا يجب أن تقوم بنسخ نوع النموذج مباشرة. بدلاً من ذلك ، استخدم طريقة ``()createForm``. هذه الطريقة هي جزء من ``AbstractController`` وتسهل إنشاء النماذج.

.. index::
    single: Twig;form

عند تمرير نموذج إلى ال Template، استخدم `` ()createView  `` لتحويل البيانات إلى تنسيق مناسب لل Templates.

يمكن عرض النموذج في ال Template عن طريق وظيفة `` النموذج `` ل Twig:

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

عند تحديث صفحة مؤتمر (Conference) في المتصفح ، لاحظ أن كل حقل نموذج يعرض ال Widget HTML الصحيح (يتم اشتقاق نوع البيانات من النموذج):

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

تنشئ وظيفة `` ()form  `` نموذج HTML بناءً على جميع المعلومات المحددة في نوع النموذج. كما تضيف أيضًا `` enctype = multipart / form-data `` على علامة `` <form> `` كما هو مطلوب في حقل إدخال تحميل الملف (File upload input field ). بالإضافة إلى ذلك ، فإنها تعتني بعرض الأخطاء عندما يحتوي الإرسال على بعضها. يمكن تخصيص كل شيء عن طريق تجاوز القوالب الافتراضية(default templates) ، لكننا لن نحتاجها لهذا المشروع.

تخصيص نوع النموذج
--------------------------------

حتى إذا تم تكوين حقول النموذج استنادًا إلى نظير الميدان (Model) الخاص بها ، يمكنك تخصيص الترتيب الافتراضي (Default configuration) في فئة نوع النموذج مباشرة:

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

لاحظ أننا أضفنا زر إرسال (Submit) (يسمح لنا بالاستمرار في استخدام تعبير`` {{(form (comment_form)}} `` البسيط في النموذج).

لا يمكن ترتيب بعض الحقول تلقائيًا ، مثل حقل ``photoFilename``. يحتاج كيان ``التعليق`` فقط إلى حفظ اسم ملف الصورة ، ولكن يجب أن يتعامل النموذج مع تحميل الملف نفسه. للتعامل مع هذه الحالة ، قمنا بإضافة حقل يسمى حقل ``الصورة / photo`` على أنه ``غير محدد un-mapped``: لن يتم تعيينه لأي خاصية في ``التعليق / Comment``. سنقوم بإدارته يدويًا لتنفيذ منطق معين (مثل تخزين الصورة التي تم تحميلها على القرص).

كمثال للتخصيص ، قمنا أيضًا بتعديل التسمية الافتراضية لبعض الحقول.

قيد الصورة يعمل بالتحقق من نوع mime ؛ تتطلب مكون Mime لجعله يعمل:

.. code-block:: terminal

    $ symfony composer req mime

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

التحقق من الميادين (Models)
-------------------------------------------

يقوم نوع النموذج بترتيب عرض الواجهة الأمامية للنموذج (عبر HTML5 validation). فيما يلي نموذج HTML الذي تم إنشاؤه:

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

يستخدم النموذج إدخال ``البريد الإلكتروني / email`` للبريد الإلكتروني للتعليق ويجعل معظم الحقول `` مطلوبة ``. لاحظ أن النموذج يحتوي أيضًا على حقل مخفي لـ ``_token`` لحماية النموذج من `هجمات <https://owasp.org/www-community/attacks/csrf>`_.

ولكن إذا تجاوز إرسال النموذج التحقق من HTML (باستخدام عميل HTTP لا يفرض قواعد التحقق من الصحة مثل cURL) ، فقد تصل البيانات غير الصالحة إلى الخادم.

نحتاج أيضًا إلى إضافة بعض قيود التحقق من الصحة إلى نموذج بيانات`` التعليق / Comment``:

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

التعامل مع نموذج
------------------------------

الكود الذي كتبناه حتى الآن يكفي لعرض النموذج.

يجب علينا الآن معالجة إرسال النموذج وتسجيل  معلوماته في قاعدة البيانات في وحدة التحكم:

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

عندما يتم إرسال النموذج ، يتم تحديث كائن `` التعليق / Comment `` وفقًا للبيانات المقدمة.

يُضطر المؤتمر إلى أن يكون هو نفسه الذي أتى من عنوان URL (قمنا بإزالته من النموذج).

إذا كان النموذج غير صالح ، فنحن نعرض الصفحة ، ولكن النموذج سيحتوي الآن على القيم المرسلة ورسائل الخطأ بحيث يمكن عرضها مرة أخرى للمستخدم.

جرب النموذج. يجب أن يعمل بشكل جيد ويجب تخزين البيانات في قاعدة البيانات (تحقق من ذلك في الواجهة الخلفية للمشرف). هناك مشكلة واحدة: الصور. إنها لا تعمل لأننا لم نتعامل معها بعد في وحدة التحكم.

تحميل الملفات
-------------------------

يجب تخزين الصور التي تم تحميلها على القرص المحلي ، في مكان يمكن الوصول إليه من خلال الواجهة الأمامية حتى نتمكن من عرضها على صفحة المؤتمر. سنقوم بتخزينها تحت دليل ``public/uploads/photos``:

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

لإدارة تحميل الصور ، نقوم بإنشاء اسم عشوائي للملف. ثم ننقل الملف الذي تم تحميله إلى موقعه النهائي (دليل الصور). أخيرًا ، نقوم بتخزين اسم الملف في كائن التعليق.

.. index::
    single: Container;Bind
    single: Bind

لاحظ الخاصية الجديدة في طريقة ``()show``؟ ``photoDir$`` عبارة عن string  وليست خدمة. كيف يمكن لـ Symfony معرفة ما يجب حقنه هنا؟ تستطيع حاوية Symfony تخزين *المعلمات / parameters* بالإضافة إلى الخدمات. المعلمات هي مقاييس تساعد في تكوين الخدمات. يمكن إدخال هذه المعلمات في الخدمات بشكل صريح ، أو يمكن *ربطها بالاسم*:

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

يسمح إعداد ``bind`` لـ Symfony بإدخال القيمة كلما كانت للخدمة خاصية ``photoDir$``.

حاول تحميل ملف PDF بدلاً من الصورة. يجب أن ترى رسائل الخطأ . التصميم قبيح للغاية في الوقت الحالي ، ولكن لا تقلق ، كل شيء سيصبح جميلًا في بضع خطوات عندما سنعمل على تصميم الموقع. بالنسبة للنماذج ، سنقوم بتغيير سطر واحد من التكوين لنمط جميع عناصر النموذج.

علاج وتصحيح النماذج
------------------------------------

عندما يتم إرسال نموذج لا يعمل  بشكل جيد ، استخدم لوحة "النموذج" في Symfony Profiler. يمنحك معلومات حول النموذج وخياراته والبيانات المقدمة وكيفية تحويلها داخليًا. إذا احتوى النموذج على أية أخطاء ، فسيتم سردها أيضًا.

يسير العمل النموذجي على النحو التالي:

* يتم عرض النموذج على صفحة ؛

* يقدم المستخدم النموذج عبر طلب POST ؛

* يقوم الخادم بإعادة توجيه المستخدم إلى صفحة أخرى أو إلى نفس الصفحة.

ولكن كيف يمكنك الوصول إلى ال Profiler يخص طلب إرسال ناجح؟ نظرًا لأنه تتم إعادة توجيه الصفحة على الفور ، لا نرى شريط أدوات تصحيح الويب (web debug toolbar) لطلب POST. هذا ليس بمشكل: في الصفحة المعاد توجيهها ، مرر مؤشر الماوس فوق الجزء الأخضر "200" الأيسر. يجب أن تشاهد إعادة التوجيه "302" مع ارتباط إلى ملف التعريف (بين قوسين).

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

انقر عليه للوصول إلى ملف طلب POST ، وانتقل إلى لوحة "النموذج / Form":

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

عرض الصور التي تم تحميلها في الخلفية الإدارية
-----------------------------------------------------------------------------------

تعرض الواجهة الخلفية للمسؤول حاليًا اسم ملف الصورة ، لكننا نريد أن نرى الصورة الفعلية:

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

إستبعاد الصور التي تم تحميلها من Git
---------------------------------------------------------------

لا تنفذ commit بعد! لا نريد تخزين الصور التي تم تحميلها في مستودع Git. أضف دليل ``/public/uploads`` إلى ملف ``.gitignore`` :

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

تخزين الملفات التي تم تحميلها على خوادم الإنتاج (Production Servers)
------------------------------------------------------------------------------------------------------------

الخطوة الأخيرة هي تخزين الملفات التي تم تحميلها على خوادم الإنتاج. لماذا يجب علينا القيام بشيء خاص؟ لأن معظم منصات السحابة الحديثة تستخدم حاويات للقراءة فقط لأسباب مختلفة. Upsun ليست استثناء.

ليس كل شيء للقراءة فقط في مشروع Symfony. نحاول جاهدين إنشاء أكبر قدر ممكن من ذاكرة التخزين المؤقت عند بناء الحاوية (خلال مرحلة تدفئة ذاكرة التخزين المؤقت) ، ولكن لا تزال Symfony بحاجة إلى الكتابة في مكان ما لذاكرة التخزين المؤقت للمستخدم ، والسجلات ، والجلسات إذا تم تخزينها على نظام الملفات، واكثر.

ألق نظرة على  ``symfony.cloud.yaml.`` ، يوجد بالفعل *جبل قابل للكتابة* للدليل ``/var``. الدليل ``/var`` هو الدليل الوحيد الذي تكتب فيه Symfony (ذاكرة التخزين المؤقت ، السجلات ، ...).

لنقم بإنشاء حامل جديد للصور التي تم تحميلها:

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

يمكنك الآن نشر الكود وسيتم تخزين الصور في دليل `` /public/uploads `` مثل نسختنا المحلية.

.. sidebar:: الذهاب أبعد من ذلك

    * `البرنامج التعليمي لـ SymfonyCasts Forms <https://symfonycasts.com/screencast/symfony-forms>`_؛

    * كيفية `تخصيص عرض نموذج Symfony في HTML <https://symfony.com/doc/current/form/form_customization.html>`_؛

    * `التحقق من نماذج Symfony <https://symfony.com/doc/current/forms.html#validating-forms>`_؛

    * `مرجع أنواع نماذج Symfony <https://symfony.com/doc/current/reference/forms/types.html>`_؛

    * `مستندات <https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md>`_, والتي توفر التكامل مع العديد من موفري التخزين السحابي ، مثل AWS S3 ، Azure و Google Cloud Storage؛

    * `معلمات إعداد Symfony <https://symfony.com/doc/current/configuration.html#configuration-parameters>`_.

    * `قيود المصادقة ل Symfony <https://symfony.com/doc/current/validation.html#basic-constictions>`_؛

    * `سيمفوني ورقة غش نموذج <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf>`_.

.. _`نوع النموذج / form type`: https://symfony.com/doc/current/forms.html#form-types
