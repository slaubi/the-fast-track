زیباسازی رابط کاربری با Webpack
===================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

ما هیچ زمانی را صرف طراحی رابط کاربری نکرده‌ایم. برای زیباسازی حرفه‌ای، ما از یک پشته‌ی مدرن و مبتنی بر `Webpack <https://webpack.js.org/>`_ استفاده می‌کنیم. بیایید برای افزودن حال‌وهوای سیمفونی و تسهیل ادغام با اپلیکیشن، *Webpack Encore* را نصب کنیم:

.. code-block:: bash

    $ symfony composer req encore

یک محیط کامل Webpack برای شما ایجاد شده است: ``package.json`` و ``webpack.config.js`` تولید شده حاوی پیشفرض‌های خوبی است. ``webpack.config.js`` را باز کنید، این فایل از انتزاعی‌سازی Encore برای پیکربندی Webpack بهره می‌برد.

فایل ``package.json`` تعدادی فرمان دلپذیر تعریف می‌کند که همیشه از آن‌ها استفاده خواهیم کرد.

The ``assets`` directory contains the main entry points for the project assets: ``styles/app.css`` and ``app.js``.

استفاده از Sass
------------------------

.. index::
    single: Sass

به جای استفاده از CSS خام، بیایید از `Sass <https://sass-lang.com/>`_ بهره بگیریم:

.. code-block:: bash

    $ mv assets/styles/app.css assets/styles/app.scss

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -6,7 +6,7 @@
      */

     // any CSS you import will output into a single css file (app.css in this case)
    -import './styles/app.css';
    +import './styles/app.scss';

     // Need jQuery? Install it with "yarn add jquery", then uncomment to import it.
     // import $ from 'jquery';

بارگیرنده‌ی Sass را نصب کنید:

.. code-block:: bash

    $ yarn add node-sass sass-loader --dev

و بارگیرنده‌ی Sass را در webpack فعال کنید:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -54,7 +54,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

از کجا می‌دانستم که باید چه بسته‌هایی را نصب کنم؟ اگر سعی می‌کردیم که دارایی‌ها را بدون این بسته‌ها بسازیم، Encore پیغام خطای مناسبی به ما می‌داد که پیشنهاد می‌کرد، فرمان ``yarn add`` برای نصب وابستگی‌های لازم برای بارگرفتن فایل‌های ``.scss`` الزامی است.

بهره‌گیری از Bootstrap
----------------------------------

.. index::
    single: Bootstrap

برای شروع با پیشفرض‌های خوب و ساخت یک وب‌سایت واکنشی، یک چارچوب CSS همچون `Bootstrap <https://getbootstrap.com/>`_ می‌تواند بسیار مفید باشد. آن را به عنوان یک بسته نصب کنید:

.. code-block:: bash

    $ yarn add bootstrap jquery popper.js bs-custom-file-input --dev

Bootstrap را در فایل CSS الزام (require) کنید (ما همچنین فایل را تمیزکاری کرده‌ایم):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

همین کار را برای فایل JS انجام دهید:

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -7,8 +7,7 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';

    -// Need jQuery? Install it with "yarn add jquery", then uncomment to import it.
    -// import $ from 'jquery';
    -
    -console.log('Hello Webpack Encore! Edit me in assets/app.js');
    +bsCustomFileInput.init();

سیستم فرم سیمفونی به صورت بومی از Bootstrap به همراه یک پوسته‌ی ویژه پشتیبانی می‌کند. آن را فعال کنید:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_4_layout.html.twig']

زیباسازی HTML
---------------------

حالا آماده هستیم تا اپلیکیشن را زیباسازی کنیم. فایل فشرده‌ی را بارگیری کرده و آن را در پوشه‌ی اصلی پروژه از حالت فشرده خارج کنید:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-5.0.zip', 'guestbook-5.0.zip');"
    $ unzip -o guestbook-5.0.zip
    $ rm guestbook-5.0.zip

به قالب‌ها نگاهی بیاندازید، ممکن است یکی‌دو حقه در مورد Twig بیاموزید.

ساخت دارایی‌ها
----------------------------

.. index::
    single: Symfony CLI;run

یک تغییر اساسی هنگام استفاده از Webpack این است که فایل‌های CSS و JS به صورت مستقیم در اپلیکیشن قابل استفاده نیستند. آن‌ها ابتدا نیاز دارند که «compile» شوند.

در محیط توسعه، کامپایل‌کردن دارایی‌ها می‌تواند از طریق فرمان ``encore dev`` انجام شود:

.. code-block:: bash

    $ symfony run yarn encore dev

به جای اینکه با هربار تغییر، فرمان را اجرا کنیم، آن را به پس‌زمینه بفرستید و اجازه دهید تا تغییرات JS و CSS را دنبال کند:

.. code-block:: bash
    :class: ignore

    $ symfony run -d yarn encore dev --watch

برای کشف تغییرات بصری، زمان بگذارید. در مرورگر نگاهی به طراحی جدید داشته باشید.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

حالا فرم ورود به سایتِ تولیده‌شده، زیباسازی شده است چرا که باندل Maker به صورت پیشفرض از کلاس‌های CSS موجود در Bootstrap استفاده می‌کند:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

در محیط عمل‌آوری، SymfonyCloud به صورت خودکار تشخیص می‌دهد که شما از Encore استفاده می‌کنید و در طول فاز ساخت، دارایی‌ها را برای شما کامپایل می‌کند.

.. sidebar:: بیشتر بدانید

    * `مستندات Webpack <https://webpack.js.org/concepts/>`_؛

    * `مستندات سیمفونی Webpack Encore <https://symfony.com/doc/current/frontend.html>`_؛

    * `آموزش تصویری Webpack Encore در SymfonyCasts <https://symfonycasts.com/screencast/webpack-encore>`_.
