Стилізація інтерфейсу користувача за допомогою Webpack
================================================================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

Ми не витрачали часу на дизайн інтерфейсу користувача. Щоб стилізувати як професіонали, ми будемо використовувати сучасний стек, заснований на `Webpack`_. А щоб додати штрих Symfony і полегшити його інтеграцію із застосунком, використовуймо *Webpack Encore*:

.. code-block:: terminal

    $ symfony composer req encore

Для вас створено повноцінне середовище Webpack: ``package.json`` і ``webpack.config.js`` згенеровані й містять готову до використання конфігурацію за замовчуванням. Відкрийте ``webpack.config.js``, він використовує абстракцію Encore для налаштування Webpack.

Файл ``package.json`` визначає кілька зручних команд, які ми будемо використовувати постійно.

Каталог ``assets`` містить основні точки входу для ресурсів проекту: ``styles/app.css`` та ``app.js``.

Використання Sass
-----------------------------

.. index::
    single: Sass

Замість того щоб використовувати звичайний CSS, перемкнімося на `Sass`_:

.. code-block:: terminal

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

     // start the Stimulus application
     import './bootstrap';

Встановіть завантажувач Sass:

.. code-block:: terminal

    $ npm install node-sass sass-loader --save-dev

І увімкніть його у webpack:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -57,7 +57,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

Як я дізнався, які пакети потрібно встановити? Якби ми намагалися зібрати наші ресурси без них, Encore видав би нам приємне повідомлення про помилку, пропонуючи команду ``npm install``, необхідну для встановлення залежностей, щоб завантажити файли ``.scss``

Використання Bootstrap
----------------------------------

.. index::
    single: Bootstrap

Щоб почати з хороших налаштувань за замовчуванням і створити адаптивний веб-сайт, CSS-фреймворк, на зразок `Bootstrap`_, може значною мірою стати в пригоді. Встановіть його як пакет:

.. code-block:: terminal

    $ npm install bootstrap @popperjs/core bs-custom-file-input --save-dev

Включіть Bootstrap у файлі CSS (ми також очистили файл):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

Зробіть те саме для файлу JS:

.. code-block:: diff
    :caption: patch_file

    --- a/assets/app.js
    +++ b/assets/app.js
    @@ -7,6 +7,10 @@

     // any CSS you import will output into a single css file (app.css in this case)
     import './styles/app.scss';
    +import 'bootstrap';
    +import bsCustomFileInput from 'bs-custom-file-input';

     // start the Stimulus application
     import './bootstrap';
    +
    +bsCustomFileInput.init();

Система форм Symfony з коробки підтримує Bootstrap зі спеціальною темою, увімкніть її:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Стилізація HTML
-------------------------

Зараз ми готові стилізувати застосунок. Завантажте та розгорніть архів у корені проекту:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Погляньте на шаблони, ви могли б дізнатися одну чи дві хитрощі про Twig.

Збірка ресурсів
-----------------------------

.. index::
    single: Symfony CLI;run

Однією з основних змін при використанні Webpack є те, що CSS і JS-файли не використовуються застосунком безпосередньо. Спочатку їх потрібно "скомпілювати".

У процесі розробки збірка ресурсів може бути виконана за допомогою команди ``encore dev``:

.. code-block:: terminal

    $ symfony run npm run dev

Замість того щоб виконувати команду щоразу, коли відбувається зміна, відправте її у фоновий режим і дозвольте спостерігати за змінами JS і CSS:

.. code-block:: terminal
    :class: ignore

    $ symfony run -d npm run watch

Знайдіть час, щоб дослідити візуальні зміни. Погляньте на новий дизайн у браузері.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Згенеровану форму входу тепер також стилізовано, оскільки бандл Maker, за замовчуванням, використовує CSS-класи Bootstrap:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

Для продакшн Upsun автоматично виявляє, що ви використовуєте Encore, і компілює ресурси для вас під час фази збірки.

.. sidebar:: Йдемо далі

    * `Документація по Webpack`_;

    * `Документація по Symfony Webpack Encore`_;

    * `Навчальний посібник SymfonyCasts: Encore`_.

.. _`Webpack`: https://webpack.js.org/
.. _`Sass`: https://sass-lang.com/
.. _`Bootstrap`: https://getbootstrap.com/
.. _`Документація по Webpack`: https://webpack.js.org/concepts/
.. _`Документація по Symfony Webpack Encore`: https://symfony.com/doc/current/frontend.html
.. _`Навчальний посібник SymfonyCasts: Encore`: https://symfonycasts.com/screencast/webpack-encore
