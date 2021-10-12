Работа с ветками
==============================

Существует множество способов организации работы с кодом в проекте. Однако фиксировать изменения непосредственно в master-ветке Git и напрямую развёртывать код в продакшен без тестирования, пожалуй, не лучший вариант.

Тестирование — это не только модульные или функциональные тесты, но и проверка поведения приложения с использованием реальных данных. Если вы или ваши `заинтересованные стороны`_ можете просмотреть приложение в том самом виде, в котором оно предстанет перед конечными пользователями, это становится огромным преимуществом и позволяет вам развёртывать приложения с уверенностью. Особенно полезно, когда нетехнические специалисты могут проверять новую функциональность.

Для упрощения и чтобы избежать повторения на следующих шагах мы продолжим работу в master-ветке Git, однако давайте посмотрим, как это можно организовать получше.

Организация рабочего процесса с помощью Git
------------------------------------------------------------------------------

Создание отдельной ветки на каждую новую функциональность или исправление бага — один из простых и эффективных вариантов организации рабочего процесса.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Создание веток
---------------------------

.. index::
    single: Git;branch
    single: Git;checkout

Рабочий процесс начинается с создания ветки в Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Команда из примера создаёт ветку ``sessions-in-redis`` из ветки ``master``.  Это действие создаёт ответвление кода и конфигурации инфраструктуры.

Хранение сессий в Redis
--------------------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Вероятно, вы уже поняли из названия ветки, что для хранения сессий вместо файловой системы мы будем использовать хранилище в Redis.

Необходимые шаги для реализации этого вполне типичны:

#. Создайте новую ветку в Git;

#. Обновите конфигурацию Symfony, если потребуется;

#. При необходимости напишите и/или обновите код;

#. Обновите конфигурацию PHP, добавив PHP-модуль Redis;

#. Обновите инфраструктуру Docker и SymfonyCloud, добавив сервис Redis;

#. Протестируйте локально;

#. Протестируйте удалённо;

#. Выполните слияние ветки с основной веткой;

#. Разверните в продакшене;

#. Удалите ветку.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Протестируйте сайт на своём компьютере, просматривая различные страницы. Поскольку мы не вносили никаких визуальных изменений и пока не используем сессии, всё должно работать как и раньше.

Развёртывание ветки
-------------------------------------

.. index::
    single: SymfonyCloud;Environment

Перед развёртыванием в продакшене, мы должны протестировать ветку в окружении ему идентичному. Нам необходимо убедиться, что всё работает корректно в Symfony-окружении ``prod`` (локальный сайт использует окружение ``dev``).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Теперь давайте создадим *SymfonyCloud-окружение* на основе *Git-ветки*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Данная команда создаёт новое окружение в следующем порядке:

* Ветка наследует код и инфраструктуру от текущей Git ветки (``sessions-in-redis``);

* Данные поступают из основного окружения (т.е. продакшена) путём создания последовательных слепков всех служебных данных, включая файлы (например, загруженные пользователями) и базы данных;

* Создаётся новый выделенный кластер для развёртывания кода, данных и инфраструктуры.

Поскольку развёртывание проходит те же этапы, что и развёртывание в продакшене, миграции базы данных также будут выполнены. Это отличный способ проверить, что миграции работают с данными в продакшене.

Окружения, отличные от ``master``, на самом деле очень на него похожи, но есть небольшие отличия. Например, отправка электронных писем отключена по умолчанию.

.. index::
    single: Symfony CLI;open:remote

После завершения развёртывания откройте новую ветку в браузере:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Обратите внимание, что все команды SymfonyCloud работают в текущей Git-ветке. Выполненная выше команда откроет URL-адрес для только что развёрнутой ветки ``sessions-in-redis``. Адрес будет выглядеть примерно так: ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Протестируйте сайт в новом окружении: вы увидите те же данные, что и в основном окружении.

Если добавить новые конференции в основное окружение (``master``), они не появятся в окружении ``sessions-in-redis`` и наоборот, так как окружения являются независимыми и полностью изолированными друг от друга.

Если код изменится в основной ветке, вы всегда можете выполнить перебазирование Git-ветки и развернуть обновлённую версию, разрешив конфликты в коде и инфраструктуре.

.. index::
    single: Symfony CLI;env:sync

В том числе вы можете синхронизировать данные с основного окружения в окружение ``sessions-in-redis``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Предварительная отладка развёртывания в продакшене
------------------------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

По умолчанию все SymfonyCloud-окружения используют те же настройки, что и окружение ``master``/``prod`` (то же, что и  окружение ``prod`` в Symfony). Это позволяет протестировать приложение в реальных условиях (как и в продакшене). Таким образом создаётся ощущение разработки и тестирования непосредственно на продакшен-серверах, но без связанных с этим рисков. Мне напоминает это старые добрые времена, когда мы развёртывали проекты через FTP.

.. index::
    single: Symfony CLI;env:debug

При возникновении проблемы вы можете переключиться на Symfony-окружение ``dev``:

.. code-block:: bash

    $ symfony env:debug

Как только закончите, переключитесь обратно к продакшен-настройкам:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Никогда** не активируйте окружение ``dev`` и никогда не используйте профилировщик Symfony, находясь в ветке ``master``, это приведёт к тому, что ваше приложение станет очень медленным и откроет множество серьёзных дыр безопасности.

Предварительное тестирование развёртывания в продакшене
----------------------------------------------------------------------------------------------------------

Возможность предварительного просмотра будущей версии сайта с продакшен-данными открывает широкие возможности: от визуального регрессионного тестирования до тестирования производительности. `Blackfire <https://blackfire.io>`_ — идеальный инструмент для такого рода задач.

Перейдите к шагу "Производительность", чтобы узнать больше про использование Blackfire для тестирования кода перед развёртыванием.

Развёртывание в продакшене
--------------------------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Если вы полностью удовлетворены изменениями в ветке с новой функциональностью, выполните слияние кода и инфраструктуры в master-ветку в Git:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

И разверните:

.. code-block:: bash

    $ symfony deploy

При развёртывании в SymfonyCloud отправляются только изменения в коде и инфраструктуре; данные останутся такими же, как и были до развёртывания.

Очистка
--------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

В завершение выполните очистку, удалив Git-ветку и SymfonyCloud-окружение:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Двигаемся дальше

    * `Ветвление в Git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`заинтересованные стороны`: https://en.wikipedia.org/wiki/Project_stakeholder
