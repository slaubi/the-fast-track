إدارة دورة حياة  Doctrine Objects
==============================================

عند إنشاء تعليق جديد ، سيكون أمر رائع أن يتم تعيين تاريخ `` createAt `` تلقائيًا بقيمة التاريخ والوقت الحاليين.

لدى Doctrine طرق مختلفة للتعامل مع الكائنات وخصائصها أثناء دورة حياتها (قبل إنشاء الصف في قاعدة البيانات ، بعد تحديث الصف ، ...).

تحديد الاسترجاعات لدورة الحياة
---------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

عندما لا يحتاج السلوك إلى أي خدمة ويجب تطبيقه على نوع واحد فقط من الكيان ، حدد رد الاتصال (callback) في فئة الكيان:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\ORM\Mapping as ORM;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    + * @ORM\HasLifecycleCallbacks()
      */
     class Comment
     {
    @@ -106,6 +107,14 @@ class Comment
             return $this;
         }

    +    /**
    +     * @ORM\PrePersist
    +     */
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTime();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

يتم تشغيل *حدث* ``ORM \ PrePersist``  عندما يتم تخزين الكائن في قاعدة البيانات لأول مرة. عند حدوث ذلك ، يتم استدعاء طريقة ``()setCreatedAtValue`` ويتم استخدام التاريخ والوقت الحاليين كقيمة لخاصية ``createdAt``.

إضافة البزاقات إلى المؤتمرات
-----------------------------------------------------

عناوين URL للمؤتمرات ليست ذات معنى: ``Conference/1/``. والأهم أنها تعتمد على تفاصيل التنفيذ (المفتاح الأساسي في قاعدة البيانات مسرب).

ماذا عن استخدام عناوين URL مثل ``/conference/paris-2020`` بدلاً من ذلك؟ هذا سيبدو أفضل بكثير. ``paris-2020`` هو ما نسميه المؤتمر *slug*.

.. index::
    single: Command;make:entity

قم بإضافة خاصية `` slug `` جديدة للمؤتمرات (سلسلة غير قابلة للإلغاء من 255 حرفًا):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

قم بإنشاء ملف ترحيل لإضافة العمود الجديد:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

وتنفيذ هذا الترحيل الجديد:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

هل حصلت على خطأ؟ هذا أمر متوقع. لماذا ا؟ لأننا طلبنا ألا يكون slug `` null `` ولكن الإدخالات الموجودة في قاعدة بيانات المؤتمر ستحصل على قيمة `` null `` عند تشغيل الترحيل. لنصلح ذلك من خلال تعديل الترحيل:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema): void

الحيلة هنا هي إضافة العمود والسماح له بأن يكون ``null`` ، ثم قم بتعيين slug إلى قيمة ليست ``null`` ، وأخيرًا ، قم بتغيير عمود slug لعدم السماح بـ ``null``.

.. note::

    بالنسبة لمشروع حقيقي ، قد يكون استخدام ``CONCAT(LOWER(city), '-', year)`` غير كافٍ. في هذه الحالة ، نحتاج إلى استخدام الSlugger "الحقيقي".

.. index::
    single: Command;doctrine:migrations:migrate

يجب أن يتم الترحيل بشكل جيد الآن:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

نظرًا لأن التطبيق سيستخدم slugs قريبًا للعثور على كل مؤتمر ، فلنقم بتعديل كيان المؤتمر للتأكد من أن قيم slug فريدة في قاعدة البيانات:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,9 +6,11 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    + * @UniqueEntity("slug")
      */
     class Conference
     {
    @@ -40,7 +42,7 @@ class Conference
         private $comments;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, unique=true)
          */
         private $slug;

.. index::
    single: Components;Validator

نظرًا لأننا نستخدم مدققًا لضمان التفرد في الاسم القصير، نحتاج إلى إضافة مكون Symfony Validator:

.. code-block:: terminal

    $ symfony composer req validator

.. index::
    single: Command;make:migration

كما كنت قد خمنت ، نحتاج إلى أداء رقصة الترحيل:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

توليد البزاقات
---------------------------

.. index::
    single: Components;String
    single: Slug

يعد إنشاء سبيكة تقرأ جيدًا في عنوان URL (حيث يجب ترميز أي شيء بخلاف أحرف ASCII) مهمة صعبة ، خاصة للغات الأخرى غير الإنجليزية. كيف يمكنك تحويل "é" إلى "e" على سبيل المثال؟

بدلاً من إعادة اختراع العجلة ، دعنا نستخدم مكون  ``String`` ل Symfony، الذي يخفف من معالجة السلاسل strings  ويوفر *slugger*:

.. code-block:: terminal

    $ symfony composer req string

أضف طريقة `` ()computeSlug `` إلى فئة `` المؤتمر `` التي تحسب البزاق بناءً على بيانات المؤتمر:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    @@ -61,6 +62,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger)
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

لا تحسب طريقة `` ()computeSlug `` بزاق إلا إذا كان السلق الحالي فارغًا أو تم تعيينه على القيمة "-" الخاصة. لماذا نحتاج إلى القيمة الخاصة "-"؟ لأنه عند إضافة مؤتمر في الخلفية ، يكون البزاق مطلوبًا. لذا ، نحتاج إلى قيمة غير فارغة تخبر التطبيق أننا نريد إنشاء بزاق بشكل تلقائي.

تحديد استدعاء دورة حياة معقدة
------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

بالنسبة لخاصية ``createdAt``، يجب ضبط ``slug`` تلقائيًا عندما يتم تحديث المؤتمر عن طريق استدعاء طريقة ``()computeSlug``.

ولكن نظرًا لأن هذه الطريقة تعتمد على تطبيق ``SluggerInterface``، لا يمكننا إضافة حدث ``prePersist`` كما كان من قبل (ليس لدينا طريقة لإدخال slugger).

بدلاً من ذلك ، قم بإنشاء مستمع كيان ل Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        private $slugger;

        public function __construct(SluggerInterface $slugger)
        {
            $this->slugger = $slugger;
        }

        public function prePersist(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }
    }

لاحظ أن الslug يتم تحديثه عند إنشاء مؤتمر جديد (``()prePersist``) وكلما تم تحديثه (``()preUpdate``).

تكوين الخدمة في الحاوية
-------------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

حتى الآن ، لم نتحدث عن أحد المكونات الرئيسية في Symfony ، *حاوية حقن التبعية (dependency injection container)*. الحاوية مسؤولة عن إدارة *الخدمات*: إنشائها وحقنها عند الحاجة.

*الخدمة* عبارة عن كائن "عام (global)" يوفر ميزات (على سبيل المثال ، مرسل بريد أو مسجّل أو سلوجر وما إلى ذلك) على عكس *كائنات البيانات (data objects)* (مثل مثيلات الكيان ل Doctrine).

نادرًا ما تتفاعل مع الحاوية مباشرة حيث تقوم تلقائيًا بإدخال كائنات الخدمة كلما احتجت إليها: تقوم الحاوية بحقن كائنات وسيطة وحدة التحكم عندما تكتب تلميحًا لها على سبيل المثال.

إذا تساءلت عن كيفية تسجيل مستمع الأحداث في الخطوة السابقة ، فلديك الجواب الآن: الحاوية. عندما يقوم الفصل بتنفيذ بعض الواجهات المحددة ، تعرف الحاوية أنه يجب تسجيل الفصل بطريقة معينة.

لسوء الحظ ، لا يتم توفير التشغيل الآلي لكل شيء ، خاصة third-party packages . مستمع الكيان الذي كتبناه للتو هو أحد الأمثلة على ذلك ؛ لا يمكن إدارتها بواسطة حاوية خدمة ل Symfony تلقائيًا لأنها لا تطبق أي واجهة ولا توسع "فئة معروفة جيدًا".

نحتاج إلى الإعلان عن المستمع جزئيًا في الحاوية. يمكن حذف أسلاك التبعية حيث لا يزال من الممكن تخمينها بواسطة الحاوية ، ولكننا نحتاج إلى إضافة بعض العلامات * يدويًا لتسجيل المستمع مع مرسل حدث Doctrine:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -29,3 +29,7 @@ services:

         # add more service definitions when explicit configuration is needed
         # please note that last definitions always *replace* previous ones
    +    App\EntityListener\ConferenceEntityListener:
    +        tags:
    +            - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
    +            - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}

.. note::

    لا تخلط بين مستمعي حدث Doctrine  و Symfony. حتى لو بدوا متشابهين جدًا ، فإنهم لا يستخدمون نفس البنية التحتية تحت الغطاء.

استخدام البزاقات في التطبيق
---------------------------------------------------

حاول إضافة المزيد من المؤتمرات في الخلفية وتغيير المدينة أو سنة مؤتمر معروف ؛ لن يتم تحديث slug إلا إذا كنت تستخدم القيمة "-" الخاصة.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;Route

التغيير الأخير هو تحديث وحدات التحكم والقوالب لاستخدام المؤتمر `` slug `` بدلاً من `` id `` الخاص بالمؤتمر للمسارات:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
             ]));
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    +    #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -18,7 +18,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -22,10 +22,10 @@
             {% endfor %}

             {% if previous >= 0 %}
    -            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
             {% endif %}
             {% if next < comments|length %}
    -            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
             {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>

يجب أن يتم الوصول إلى صفحات المؤتمر الآن من خلال البزاق الخاص به:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: الذهاب أبعد من ذلك

    * `نظام أحداث Doctrine <https://symfony.com/doc/current/doctrine/events.html>`_ (استدعاءات دورة الحياة والمستمعين ، ومستمعي الكيانات ومشتركي دورة الحياة)؛

    * `مستندات مكون String  <https://symfony.com/doc/current/components/string.html>`_؛

    * `حاوية الخدمة Service container  <https://symfony.com/doc/current/service_container.html>`_؛

    * ال `Symfony Services Cheat Sheet <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_.
