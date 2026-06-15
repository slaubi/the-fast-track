Розгалуження коду
=================================

Існує багато способів організації робочого процесу внесення змін до коду в проекті. Але робота безпосередньо в головній гілці Git та розгортання в продакшн без тестування — мабуть, не найкращий.

Тестування — це не тільки модульні чи функціональні тести, це також перевірка поведінки застосунку з продакшн даними. Якщо ви чи `зацікавлені сторони`_ можете переглянути застосунок точно так, як він буде розгорнутий для кінцевих користувачів, це стає величезною перевагою і дозволяє вам розгортати застосунок з упевненістю. Це особливо ефективно, коли люди без технічних навичок можуть перевірити нові функції.

Ми продовжимо виконувати всю роботу в основній гілці Git на наступних кроках задля простоти й щоб не повторюватися, але подивімося, як це може працювати краще.

Організація робочого процесу Git
----------------------------------------------------------

Однією з можливих варіацій робочого процесу є створення однієї гілки для реалізації нової функціональності або виправлення помилок. Це просто та ефективно.

Створення гілок
-----------------------------

.. index::
    single: Git;branch
    single: Git;checkout

Робочий процес починається зі створення гілки Git:

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

Ця команда створює гілку ``sessions-in-db`` із гілки ``master``. Це "розгалужує" код і конфігурацію інфраструктури.

Зберігання сесій у базі даних
------------------------------------------------------

.. index::
    single: Session;Database

Як ви могли здогадатися з назви гілки, ми хочемо перемкнути механізм зберігання сесій із файлової системи до сховища бази даних (тут наша база даних PostgreSQL).

Необхідні кроки, щоб зробити це реальністю, типові:

#. Створіть гілку Git;

#. Оновіть конфігурацію Symfony, якщо це необхідно;

#. Напишіть та/або оновіть код, якщо це необхідно;

#. Update the PHP configuration if needed (like adding the PostgreSQL PHP
   extension);

#. Оновіть інфраструктуру Docker і Platform.sh, якщо це необхідно (додайте сервіс PostgreSQL);

#. Протестуйте локально;

#. Протестуйте віддалено;

#. Об'єднайте гілку з основною;

#. Розгорніть у продакшн;

#. Видаліть гілку.

Щоб зберігати сесії в базі даних, змініть конфігурацію ``session.handler_id`` так, щоб вона вказувала на DSN бази даних:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -8,7 +8,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(resolve:DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax
             storage_factory_id: session.storage.factory.native

Щоб зберігати сесії в базі даних, нам потрібно створити таблицю ``sessions``. Зробіть це за допомогою міграції Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Відредагуйте файл, щоб додати створення таблиці в методі ``up()``:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,15 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs

    +        $this->addSql('
    +            CREATE TABLE sessions (
    +                sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    +                sess_data BYTEA NOT NULL,
    +                sess_lifetime INTEGER NOT NULL,
    +                sess_time INTEGER NOT NULL
    +            )
    +        ');
    +        $this->addSql('CREATE INDEX expiry ON sessions (sess_lifetime)');
         }

         public function down(Schema $schema): void

Виконайте міграцію бази даних:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Протестуйте локально, переглядаючи веб-сайт. Оскільки візуальних змін немає, і оскільки ми ще не використовуємо сесії, все має працювати як і раніше.

.. note::

    Тут нам не потрібні кроки 3-5, оскільки ми повторно використовуємо базу даних у якості сховища сесій, але розділ про використання Redis показує, наскільки просто додати, протестувати і розгорнути новий сервіс як у Docker, так і в Platform.sh.

Оскільки нова таблиця не "управляється" Doctrine, ми маємо налаштувати Doctrine так, щоб таблиця не видалялася під час наступної міграції бази даних:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/doctrine.yaml
    +++ b/config/packages/doctrine.yaml
    @@ -5,6 +5,8 @@ doctrine:
             # IMPORTANT: You MUST configure your server version,
             # either here or in the DATABASE_URL env var (see .env file)
             #server_version: '14'
    +
    +        schema_filter: ~^(?!session)~
         orm:
             auto_generate_proxy_classes: true
             naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware

Зафіксуйте свої зміни в новій гілці:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Розгортання гілки
---------------------------------

.. index::
    single: Platform.sh;Environment

Перш ніж розгорнути гілку в продакшн, ми маємо перевірити її на тій самій інфраструктурі, що і продакшн. Ми також маємо переконатися, що все добре працює в середовищі Symfony ``prod`` (локальний веб-сайт використовував середовище Symfony ``dev``).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Тепер, створімо *середовище Platform.sh* на основі *гілки Git*:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:deploy

Ця команда створює нове середовище наступним чином:

* Гілка успадковує код і інфраструктуру від поточної гілки Git  (``sessions-in-db``);

* Дані надходять з основного (продакшн) середовища шляхом створення послідовного знімка всіх службових даних, включаючи файли (наприклад, завантажені користувачем) та бази даних;

* Для розгортання коду, даних та інфраструктури створюється новий виділений кластер.

Оскільки процес розгортання слідує тим же крокам, що і розгортання в продакшн, міграції баз даних також будуть виконані. Це відмінний спосіб перевірити, що міграції працюють з продакшн даними.

Будь-яке середовище відмінне від ``master`` майже ідентичне, за винятком невеликих відмінностей: наприклад, електронні листи не надсилаються за замовчуванням.

.. index::
    single: Symfony CLI;cloud:url

Після завершення розгортання відкрийте нову гілку в браузері:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Зверніть увагу, що всі команди Platform.sh працюють у поточній гілці Git. Ця команда відкриває URL-адресу для щойно розгорнутої гілки ``sessions-in-db``; посилання матиме приблизно наступний вигляд: ``https://sessions-in-db-xxx.eu-5.platformsh.site/``.

Протестуйте веб-сайт у цьому новому середовищі, ви маєте побачити всі дані, які ви створили в основному середовищі.

Якщо ви додасте більше конференцій в середовищі ``master``, вони не будуть відображатися в середовищі ``sessions-in-db`` і навпаки. Середовища є незалежними й ізольованими.

Якщо код розвивається на основній гілці, ви завжди можете перебазувати гілку Git і розгорнути оновлену версію, вирішивши конфлікти як для коду, так і для інфраструктури.

.. index::
    single: Symfony CLI;cloud:env:sync

Ви навіть можете синхронізувати дані з основного середовища до середовища ``sessions-in-db``:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Налагодження розгортання в продакшн
-------------------------------------------------------------------

.. index::
    single: Platform.sh;Debugging

За замовчуванням всі середовища Platform.sh використовують ті ж налаштування, що й середовище ``master``/``prod`` (середовище Symfony ``prod``). Це дозволяє протестувати застосунок у реальних умовах. Таким чином створюється відчуття розробки й тестування безпосередньо на продакшн-серверах, але без пов'язаних з цим ризиків. Це нагадує мені про старі добрі часи, коли ми розгортали через FTP.

.. index::
    single: Symfony CLI;cloud:env:debug

У разі виникнення проблем, можливо, ви захочете перемкнутися на середовище Symfony ``dev``:

.. code-block:: terminal

    $ symfony cloud:env:debug

Після закінчення поверніться до продакшн налаштувань:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    **Ніколи** не вмикайте середовище ``dev`` і ніколи не вмикайте Symfony Profiler у гілці ``master``. Це зробить ваш застосунок дуже повільним і відкриє багато серйозних вразливостей у системі безпеки.

Тестування розгортання в продакшн
---------------------------------------------------------------

Наявність доступу до майбутньої версії веб-сайту з продакшн даними відкриває безліч можливостей: від візуального регресійного тестування до тестування продуктивності. `Blackfire`_ є ідеальним інструментом для роботи.

Зверніться до кроку про doc:`Продуктивність <29-performance>`, щоб дізнатися більше про те, як можна використовувати Blackfire для тестування коду перед розгортанням.

Об'єднання з продакшн
---------------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Коли вас влаштовують зміни в гілці, об'єднайте код та інфраструктуру з основною гілкою Git:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

І розгорніть:

.. code-block:: terminal

    $ symfony cloud:deploy

Під час розгортання у Platform.sh публікуються лише зміни коду й інфраструктури; на дані жодним чином це не впливає.

Очищування
--------------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Нарешті, очистьте, видаливши гілку Git і середовище Platform.sh:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Йдемо далі

    * `Розгалуження Git`_;

.. _`зацікавлені сторони`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Розгалуження Git`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
