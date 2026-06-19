عرض API مع منصة  (API Platform)
========================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

لقد انتهينا من تنفيذ موقع سجل الزوار. للسماح بمزيد من استخدام البيانات ، ماذا عن كشف API الآن؟ يمكن استخدام واجهة برمجة التطبيقات (API) بواسطة تطبيق جوال لعرض جميع المؤتمرات ، وتعليقاتهم ، وربما السماح للحاضرين بإرسال التعليقات.

في هذه الخطوة ، سنقوم بتنفيذ API للقراءة فقط.

تثبيت منصة API
-----------------------

من الممكن تعريض واجهة برمجة التطبيقات عن طريق كتابة بعض الأكواد ، ولكن إذا أردنا استخدام المعايير ، فمن الأفضل أن نستخدم حلاً يعتني بالفعل بالتحميل الثقيل. حل مثل API المنصة:

.. code-block:: bash

    $ symfony composer req api

تعريض API للمؤتمرات
---------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;Groups

بعض التعليقات التوضيحية على فئة المؤتمر هي كل ما نحتاج إليه لتكوين API

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,16 +2,25 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiResource;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
      * @UniqueEntity("slug")
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="conference:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="conference:item"}}},
    + *     order={"year"="DESC", "city"="ASC"},
    + *     paginationEnabled=false
    + * )
      */
     class Conference
     {
    @@ -20,21 +29,25 @@ class Conference
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $city;

         /**
          * @ORM\Column(type="string", length=4)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $year;

         /**
          * @ORM\Column(type="boolean")
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $isInternational;

         /**
    @@ -45,6 +58,7 @@ class Conference
         /**
          * @ORM\Column(type="string", length=255, unique=true)
          */
    +    #[Groups(['conference:list', 'conference:item'])]
         private $slug;

         public function __construct()

يقوم الشرح الرئيسي `` @ ApiResource`` بتكوين واجهة برمجة التطبيقات للمؤتمرات. يقيد العمليات المحتملة `` get``وتكوين أشياء مختلفة: مثل الحقول التي سيتم عرضها وكيفية ترتيب المؤتمرات.

بشكل افتراضي ، نقطة الدخول الرئيسية لواجهة برمجة التطبيقات هي ``/api`` بفضل التكوين من``config/routes/api_platform.yaml`` التي تمت إضافتها بواسطة وصفة الحزمة.

تسمح لك واجهة الويب بالتفاعل مع واجهة برمجة التطبيقات:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

استخدمها لاختبار الاحتمالات المختلفة:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

تخيل الوقت الذي يستغرقه تنفيذ كل هذا من الصفر!

تعريض API للتعليقات
---------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;@ApiFilter
    single: Annotations;Groups

افعل نفس الشيء للتعليقات:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,13 +2,26 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiFilter;
    +use ApiPlatform\Core\Annotation\ApiResource;
    +use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
      * @ORM\HasLifecycleCallbacks()
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="comment:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="comment:item"}}},
    + *     order={"createdAt"="DESC"},
    + *     paginationEnabled=false
    + * )
    + *
    + * @ApiFilter(SearchFilter::class, properties={"conference": "exact"})
      */
     class Comment
     {
    @@ -17,18 +30,21 @@ class Comment
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $author;

         /**
          * @ORM\Column(type="text")
          */
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $text;

         /**
    @@ -36,22 +52,26 @@ class Comment
          */
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private $email;

         /**
          * @ORM\Column(type="datetime")
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $createdAt;

         /**
          * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
          * @ORM\JoinColumn(nullable=false)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $conference;

         /**
          * @ORM\Column(type="string", length=255, nullable=true)
          */
    +    #[Groups(['comment:list', 'comment:item'])]
         private $photoFilename;

         /**

يتم استخدام نفس النوع من التعليقات التوضيحية لتكوين الفصل.

تقييد التعليقات المكشوفة بواسطة API
---------------------------------------------------------------

بشكل افتراضي ، يكشف منصة API كل الإدخالات من قاعدة البيانات. ولكن بالنسبة للتعليقات ، يجب أن تكون التعليقات المنشورة فقط جزءًا من واجهة برمجة التطبيقات.

عندما تحتاج إلى تقييد العناصر التي يتم إرجاعها بواسطة API ، قم بإنشاء خدمة تنفذ ``QueryCollectionExtensionInterface`` للتحكم في استعلام Doctrine المستخدم في المجموعات و / أو ``QueryItemExtensionInterface`` للتحكم في العناصر:

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 13-15,20-22

    namespace App\Api;

    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }
    }

تطبق فئة ملحق الاستعلام المنطق الخاص بها فقط لمورد ``Comment`` وتعديل أداة إنشاء استعلام العقيدة للنظر في التعليقات في حالة ``published`` فقط.

تكوين CORS
---------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

تبعًا للإعدادات الافتراضية ، فإن سياسة الأمان ذات الأصل الأصلي لعملاء HTTP الحديث تجعل استدعاء API من مجال آخر محظور. حزمة CORS ، المثبتة كجزء من ``composer req api``، يرسل رؤوس مشاركة المصادر المشتركة استنادًا إلى متغير البيئة `` CORS_ALLOW_ORIGIN``

تبعًا للإعدادات الافتراضية ، تسمح قيمته ، المعرّفة في `` .env`` ، بطلبات HTTP من ``localhost`` و``127.0.0.1`` على أي منفذ. هذا هو بالضبط ما نحتاجه للخطوة التالية حيث سنقوم بإنشاء SPA التي سيكون لها خادم الويب الخاص بها والذي سيتصل بـ API.

.. sidebar:: الذهاب أبعد من ذلك

    * `SymfonyCasts API Platform tutorial <https://symfonycasts.com/screencast/api-platform>`_؛

    * لتمكين دعم GraphQL ، قم بتشغيل ``composer require webonyx/graphql-php`` ، ثم استعرض للوصول إلى ``api/graphql/``.
