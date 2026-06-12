Выбор методологии разработки
======================================================

Учить — значит повторять что-то одно и то же снова и снова. Я не буду применять такой подход, обещаю. В конце каждого шага исполните победный танец и сохраните результат проделанной работы. Это подобно нажатию ``Ctrl+S``, только для всего сайта целиком.

Стратегия использования Git
-------------------------------------------------

.. index::
    single: Git;add
    single: Git;commit

В конце каждого шага не забудьте зафиксировать изменения в git:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Вы можете смело добавлять в коммит все файлы, потому что при создании Symfony-проекта уже был подготовлен файл ``.gitignore``. Кроме того, каждый пакет может добавлять свои собственные шаблоны игнорируемых файлов в ``.gitignore``. Взгляните на текущее содержимое:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Эти странные строки — маркеры, добавленные Symfony Flex, чтобы определить, что нужно удалить, если вы решите удалить зависимость. Как я упоминал ранее, вся эта утомительная работа выполняется автоматически Symfony.

Теперь самое время разместить репозиторий на удалённый сервер. Для этого прекрасно подойдёт GitHub, GitLab или Bitbucket.

.. note::

    If you are deploying on Upsun, you already have a copy of the Git repository as Upsun uses Git behind the scenes when you are using ``cloud:push``. But you should not rely on the Upsun Git repository. It is only for deployment usage. It is not a backup.

Развёртывание в продакшене с помощью непрерывной интеграции
----------------------------------------------------------------------------------------------------------------

.. index::
    single: Symfony CLI;cloud:push

Ещё одной хорошей практикой является частое развёртывание. Развёртывание в конце каждого шага — хорошо и полезно:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
