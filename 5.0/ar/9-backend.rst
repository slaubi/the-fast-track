إعداد  النظام الخلفي
=====================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

تعتبر إضافة المؤتمرات القادمة إلى قاعدة البيانات مهمة خاصة بالمشرفين. *قاعدة الإشراف الخلفية* تعتبر جناح محمي فالموقع, حيث يمكن *للمشرفين* إدارة بيانات موقع الويب وتنسيق عمليات إرسال التعليقات وغير ذلك.

كيف يمكننا بداية العمل بسرعة ؟ باستعمال أدوات لإنشاء قاعدة الإشراف الخلفية، EasyAdmin يمكننا من القيام بهذه العملية بطريقة جيدة جدا.

تهيئة EasyAdmin
--------------------

أولاً، أضف EasyAdmin للمشروع:

.. code-block:: bash

    $ symfony composer req "admin:^2"

لإعداد EasyAdmin، تم إنشاء ملف إعداد جديد عبر وصفته Flex:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

تحتوي جميع الحزم المثبتة على تهيئة مثل هذه الموجودة ضمن دليل ``config/package/``. في معظم الأوقات ، تم اختيار الإعدادات الافتراضية بعناية للعمل مع معظم التطبيقات.

قم بإزالة التعليق من الأسطر الأولى، و أضف  فئات نموذج المشروع (classes)

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

قم بالوصول الي خلفية المسؤول التي تم إنشائها في `` /admin``. بووم! واجهة إدارة لطيفة وغنية بالميزات للمؤتمرات والتعليقات:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

.. tip::

    Why is the backend accessible under ``/admin``? That's the default prefix configured in ``config/routes/easy_admin.yaml``:

    .. code-block:: yaml
        :caption: config/routes/easy_admin.yaml
        :class: ignore

        easy_admin_bundle:
            resource: '@EasyAdminBundle/Controller/EasyAdminController.php'
            prefix: /admin
            type: annotation

    You can change it to anything you like.

Adding conferences and comments is not possible yet as you would get an error: ``Object of class App\Entity\Conference could not be converted to string``. EasyAdmin tries to display the conference related to comments, but it can only do so if there is a string representation of a conference. Fix it by adding a ``__toString()`` method on the ``Conference`` class:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -44,6 +44,11 @@ class Conference
             $this->comments = new ArrayCollection();
         }

    +    public function __toString(): string
    +    {
    +        return $this->city.' '.$this->year;
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

قم بنفس الشيء للنموذج ``Comment``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -48,6 +48,11 @@ class Comment
          */
         private $photoFilename;

    +    public function __toString(): string
    +    {
    +        return (string) $this->getEmail();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

يمكنك الآن إضافة / تعديل / حذف المؤتمرات مباشرة من الواجهة الخلفية للمشرف. العب بها وأضف مؤتمرًا واحدًا على الأقل.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

أضف بعض التعليقات بدون صور. قم بوضع التاريخ يدوياً في الوقت الحالي؛ سوف نقوم بملئ خلية الـ ``createdAt`` تلقائياً في خطوة مُتقدمة.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

تخصيص EasyAdmin
--------------------

تعمل الواجهة الخلفية الافتراضية للمشرف بشكل جيد ، ولكن يمكن تخصيصها بعدة طرق لتحسين التجربة. دعونا نقوم ببعض التغييرات البسيطة لتوضيح الاحتمالات. استبدل التكوين الحالي بما يلي:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        site_name: Conference Guestbook

        design:
            menu:
                - { route: 'homepage', label: 'Back to the website', icon: 'home' }
                - { entity: 'Conference', label: 'Conferences', icon: 'map-marker' }
                - { entity: 'Comment', label: 'Comments', icon: 'comments' }

        entities:
            Conference:
                class: App\Entity\Conference

            Comment:
                class: App\Entity\Comment
                list:
                    fields:
                        - author
                        - { property: 'email', type: 'email' }
                        - { property: 'createdAt', type: 'datetime' }
                    sort: ['createdAt', 'ASC']
                    filters: ['conference']
                edit:
                    fields:
                        - { property: 'conference' }
                        - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                        - 'author'
                        - { property: 'email', type: 'email' }
                        - text

We have overridden the ``design`` section to add icons to the menu items and to add a link back to the website home page.

For the ``Comment`` section, listing the fields lets us order them the way we want. Some fields are tweaked, like setting the creation date to read-only. The ``filters`` section defines which filters to expose on top of the regular search field.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

هذه التخصيصات هي مجرد مقدمة صغيرة للإمكانيات التي يوفرها EasyAdmin.

العب مع المسؤول ، وقم بتصفية التعليقات حسب المؤتمر ، أو ابحث عن التعليقات عبر البريد الإلكتروني على سبيل المثال. المشكلة الوحيدة هي أنه يمكن لأي شخص الوصول إلى الواجهة الخلفية. لا تقلق ، سنقوم بتأمينه في خطوة مستقبلية.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: الذهاب أبعد من ذلك

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Symfony framework مرجع الإعدادات <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
