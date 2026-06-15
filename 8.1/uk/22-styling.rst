Стилізація інтерфейсу користувача
=================================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

Ми ще не приділили часу дизайну інтерфейсу користувача. Щоб стилізувати як професіонал, ми використаємо сучасний стек на основі *AssetMapper*, компонента Symfony, який керує нашими ресурсами від найпершого кроку цієї книги.

AssetMapper приймає сучасні веб-стандарти: файли JavaScript і CSS роздаються без змін і зв'язуються разом за допомогою *importmap*, дозволяючи браузеру завантажувати нативні *ES-модулі* безпосередньо. Жодного бандлера, жодного кроку збірки, жодного Node.js.

Погляньте на файл ``importmap.php`` у корені проекту: він описує пакети JavaScript, що використовуються застосунком. Функція Twig ``importmap()``, що викликається у ``templates/base.html.twig``, надає їх браузеру.

Використання Bootstrap
----------------------

.. index::
    single: Bootstrap

Щоб почати з хороших налаштувань за замовчуванням і побудувати адаптивний веб-сайт, CSS-фреймворк, як-от `Bootstrap`_, може дуже допомогти. Встановіть його як пакет importmap:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

Команда реєструє пакет у ``importmap.php`` і завантажує його (та його залежність ``@popperjs/core``) до ``assets/vendor/``; застосунок не залежить від CDN під час виконання.

Імпортуйте Bootstrap у головну точку входу JavaScript (ми також прибрали стандартне вітальне повідомлення):

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -5,6 +5,6 @@ import './stimulus_bootstrap.js';
      * This file will be included onto the page via the importmap() Twig function,
      * which should already be in your base.html.twig.
      */
    +import 'bootstrap';
    +import 'bootstrap/dist/css/bootstrap.min.css';
     import './styles/app.css';
    -
    -console.log('This log comes from assets/app.js - welcome to AssetMapper! 🎉');

Зверніть увагу, що ``app.css`` імпортується *після* стилів Bootstrap, щоб наші налаштування мали перевагу.

Система форм Symfony підтримує Bootstrap нативно за допомогою спеціальної теми, увімкніть її:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

Стилізація HTML
---------------

Тепер ми готові стилізувати застосунок. Завантажте й розпакуйте архів у корені проекту:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

Погляньте на шаблони, ви можете дізнатися про один-два трюки щодо Twig.

Роздача ресурсів
----------------

.. index::
    single: AssetMapper;asset-map:compile

Нічого збирати не потрібно: оновіть сторінку, і зміни вже діють. У середовищі розробки AssetMapper роздає файли ресурсів безпосередньо.

Приділіть час, щоб відкрити для себе візуальні зміни. Погляньте на новий дизайн у браузері.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Згенерована форма входу тепер також стилізована, оскільки Maker bundle за замовчуванням використовує CSS-класи Bootstrap:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

Для продакшену Upsun автоматично запускає команду ``asset-map:compile`` під час фази збірки: усі ресурси копіюються до ``public/assets/`` з хешем версії в іменах файлів, що уможливлює безпечне довгострокове HTTP-кешування.

.. sidebar:: Йдемо далі

    * `Документація компонента AssetMapper`_;

    * `Специфікація importmap`_;

    * `Документація Bootstrap`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`Документація компонента AssetMapper`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`Специфікація importmap`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`Документація Bootstrap`: https://getbootstrap.com/docs/
