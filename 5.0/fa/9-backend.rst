راه‌اندازی پشت صحنه‌ی مدیریتی
=========================================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

وظیفه مدیران پروژه، افزودن کنفرانس‌های آینده به پایگاه‌داده است. *پشت صحنه‌ی مدیریتی*، یک قسمت محافظت‌شده در وبسایت است که *مدیران پروژه* می‌توانند داده‌های وبسایت را مدیریت کرده، ارسال بازخوردها را تعدیل نموده و وظایفِ متعدد دیگری را انجام دهند.

چگونه می‌توانیم این بخش را به سرعت ایجاد کنیم؟ به کمک یک باندل که قادر به تولید پشت صحنه‌ی مدیریتی بر اساس مدل پروژه است. باندل EasyAdmin کاملاً برای این کار مناسب می باشد.

پیکربندی EasyAdmin
--------------------------

ابتدا EasyAdmin را به عنوان وابستگی به پروژه اضافه نمایید:

.. code-block:: bash

    $ symfony composer req "admin:^2"

برای پیکربندی EasyAdmin، یک فایل پیکربندی جدید توسط سیمفونی Flex (بر اساس recipe مربوط به باندل) تولید شده است:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

تقریباً تمام بسته‌ها مانند این بسته، دارای یک فایل پیکربندی در پوشه‌ی ``config/packages/`` هستند. اکثر اوقات، پیشفرض‌ها با دقت زیاد و به نحوی انتخاب شده‌اند که برای بیشتر اپلیکیشن‌ها کار کنند.

قسمت اول خطوط را از حالت کامنت خارج کرده و کلاس‌های مدل پروژه را به آن بیافزایید:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

وارد پشت صحنه‌ی مدیریتی ایجادشده که در آدرس ``/admin`` قرار دارد، بشوید. واو! یک رابط کاربری زیبا و پرامکانات برای کنفرانس‌ها و کامنت‌ها:

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

همین کار را برای کلاس ``Comment`` انجام دهید:

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

شما اکنون از طریق پشت صحنه‌ی مدیریتی می توانید کنفرانس‌ها را به طور مستقیم اضافه کرده، تغییر داده و یا حذف نمایید. کمی با آن بازی کنید و حداقل یک کنفرانس اضافه نمایید.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

تعدادی نظر بدون عکس اضافه کنید. فعلاً تاریخ را به صورت دستی تنظیم کنید؛ در گام بعدی، ستون ``createdAt`` را به صورت خودکار پر می‌کنیم.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

شخصی‌سازی EasyAdmin
-----------------------------

پشت صحنه‌ی مدیریتی پیشفرض به خوبی کار می‌کند، اما می‌تواند به راههای مختلفی شخصی‌سازی گردد تا تجربه‌ی بهتری را منتقل نماید. بیایید تعدادی تغییر ساده ایجاد کنیم تا امکانات موجود نشان داده شود. پیکربندی موجود را با این موارد جایگزین نمایید:

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

این شخصی‌سازی‌ها تنها بخش کوچکی از امکاناتی که EasyAdmin در اختیار می‌گذارد را معرفی می‌کنند.

کمی با بخش مدیریت بازی کنید، کامنت‌ها را بر اساس کنفرانس فیلتر کنید، یا مثلاً کامنت‌ها را بر اساس رایانامه جستجو کنید. تنها مشکل این است که هر کسی می‌تواند به پشت صحنه مدیریتی دسترسی پیدا کند. نگران نباشید، ما در گام آتی آن را امن خواهیم نمود.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: بیشتر بدانید

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `مرجع پیکربندی‌های فریمورک سیمفونی <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
