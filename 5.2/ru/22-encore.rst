Стилизация интерфейса с помощью Webpack
===================================================================

.. index::
    single: Encore
    single: Webpack
    single: Components;Encore
    single: Stylesheet

До сих пор мы совсем не занимались дизайном пользовательского интерфейса. Для оформления интерфейса, как подобает настоящему профессионалу, воспользуемся современным стеком технологий при помощи `Webpack <https://webpack.js.org/>`_. А чтобы быть ближе к Symfony и как можно проще интегрировать Webpack в наше приложение, установим *Webpack Encore*:

.. code-block:: bash

    $ symfony composer req encore

Теперь у нас есть полноценная среда для работы Webpack: сгенерированы файлы ``package.json`` и ``webpack.config.js`` с уже рабочими настройкам по умолчанию. Откройте файл ``webpack.config.js``, который использует Encore, чтобы настроить Webpack.

В файле ``package.json`` есть несколько полезных команд, которые мы будем использовать на протяжении всей книги.

Директория ``assets`` содержит главные точки входа ресурсов в проекте: ``styles/app.css`` и ``app.js``.

Использование Sass
-------------------------------

.. index::
    single: Sass

Вместо обычного CSS давайте воспользуемся `Sass <https://sass-lang.com/>`_:

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

     // start the Stimulus application
     import './bootstrap';

Установите загрузчик Sass:

.. code-block:: bash

    $ yarn add node-sass sass-loader --dev

А затем включите Sass в конфигурации webpack:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -56,7 +56,7 @@ Encore
         })

         // enables Sass/SCSS support
    -    //.enableSassLoader()
    +    .enableSassLoader()

         // uncomment if you use TypeScript
         //.enableTypeScriptLoader()

Как я узнал, какие пакеты нужно установить? При попытке собрать ресурсы без необходимых пакетов, Encore вернул бы нам сообщение об ошибке и предложил бы выполнить команду ``yarn add``, чтобы установить зависимости для обработки файлов с расширением``.scss``.

Эффективное использование Bootstrap
-----------------------------------------------------------

.. index::
    single: Bootstrap

В создании адаптивного сайта, особенно в самом начале разработки, такой CSS-фреймворк как `Bootstrap, <https://getbootstrap.com/>`_ может значительно помочь. Установите его как пакет:

.. code-block:: bash

    $ yarn add bootstrap@4 jquery popper.js bs-custom-file-input --dev

Теперь подключите Bootstrap в следующем CSS-файле (предварительно очистив его):

.. code-block:: diff
    :caption: patch_file

    --- a/assets/styles/app.scss
    +++ b/assets/styles/app.scss
    @@ -1,3 +1 @@
    -body {
    -    background-color: lightgray;
    -}
    +@import '~bootstrap/scss/bootstrap';

То же самое нужно сделать в JS-файле:

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

У форм в Symfony есть встроенная поддержка Bootstrap со специальной темой, включите её:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_4_layout.html.twig']

Стилизация HTML-шаблона
----------------------------------------

Теперь всё готово, чтобы непосредственно перейти к оформлению внешнего вида приложения. Скачайте и распакуйте архив в корневой директории проекта:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-5.2.zip', 'guestbook-5.2.zip');"
    $ unzip -o guestbook-5.2.zip
    $ rm guestbook-5.2.zip

Изучите код шаблонов, возможно, вы узнаете кое-что новое о Twig.

Сборка ресурсов
-----------------------------

.. index::
    single: Symfony CLI;run

Один важный момент при использовании Webpack — CSS- и JS-файлы не могут использоваться напрямую в приложении. Перед этим их сначала нужно "скомпилировать".

Собрать ресурсы в процессе разработки можно с помощью команды ``encore dev``:

.. code-block:: bash

    $ symfony run yarn encore dev

Чтобы не выполнять эту команду каждый раз после внесения изменений, запустите её в фоновом режиме и оставьте наблюдать за изменениями JS и CSS:

.. code-block:: bash
    :class: ignore

    $ symfony run -d yarn encore dev --watch

Остановитесь на минутку и изучите изменения внешнего вида. Посмотрите на новый дизайн в браузере.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Теперь ранее сгенерированная форма входа имеет оформление, потому что бандл Maker по умолчанию использует CSS-классы из Bootstrap:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

В продакшен-окружении SymfonyCloud автоматически определит, что в проекте используется Encore и поэтому автоматически во время сборки приложения скомпилирует все его ресурсы.

.. sidebar:: Двигаемся дальше

    * `Документация по Webpack <https://webpack.js.org/concepts/>`_;

    * `Документация по Symfony Webpack Encore <https://symfony.com/doc/current/frontend.html>`_;

    * `Учебный видеоролик по Webpack Encore на SymfonyCast <https://symfonycasts.com/screencast/webpack-encore>`_.
