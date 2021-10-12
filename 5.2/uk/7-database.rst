Налаштування бази даних
============================================

.. index::
    single: Database

Веб-сайт гостьової книги конференції потрібен для збору відгуків під час конференцій. Нам потрібно зберігати коментарі учасників конференції в постійному сховищі.

Коментар найкраще описується фіксованою структурою даних: автор, його адреса електронної пошти, текст відгуку й необов'язкове поле: фото. Тип даних, які краще за все зберігати у традиційній системі керування базою даних.

Ми будемо використовувати систему керування базою даних PostgreSQL.

Додавання PostgreSQL до Docker Compose
-------------------------------------------------

.. index::
    single: Docker;PostgreSQL

Для управління сервісами, на нашому локальному комп'ютері, ми вирішили використовувати Docker. Створіть файл ``docker-compose.yaml`` та додайте сервіс PostgreSQL:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

Це дозволить встановити сервер PostgreSQL і налаштувати деякі змінні середовища, що контролюють ім'я й облікові записи бази даних. Значення, насправді, не є важливими.

Ми також відкриваємо порт PostgreSQL (``5432``) контейнера локальному хосту. Це допоможе отримати доступ до бази даних з нашої машини.

.. note::

    Розширення ``pdo_pgsql`` мало бути встановлено разом із PHP, на попередньому кроці.

Запуск Docker Compose
---------------------------

Запустіть Docker Compose у фоновому режимі (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Зачекайте на запуск бази даних та перевірте, чи все працює правильно:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Якщо контейнери не запустились, або у стовпчику ``State`` не відображається значення ``Up``, перевірте журнали Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Доступ до локальної бази даних
--------------------------------------------------------

Використання утиліти командного рядка ``psql`` може бути корисним, час від часу. Але вам необхідно запам'ятати облікові дані та ім’я бази даних. Менш очевидно, що вам також потрібно знати локальний порт, на якому працює база даних. Docker обирає випадковий порт, щоб ви могли одночасно працювати над декількома проектами, що використовують PostgreSQL (локальний порт є частиною виводу ``docker-compose ps``).

Якщо ви запускаєте ``psql`` за допомогою Symfony CLI, вам не потрібно нічого запам'ятовувати.

Symfony CLI автоматично виявляє сервіси Docker, запущені для проекту, і виставляє змінні середовища, необхідні ``psql`` для підключення до бази даних.

.. index::
    single: Symfony CLI;run psql

Завдяки цим домовленостям, доступ до бази даних за допомогою команди ``symfony run`` —  набагато легший:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    Якщо у вас немає доступу до бінарного файлу ``psql`` на локальному хості, ви можете запустити його за допомогою ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Резервне копіювання і відновлення бази даних
-----------------------------------------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Використовуйте ``pg_dump``, щоб створити резервну копію бази даних:

.. code-block:: bash
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

І відновіть дані:

.. code-block:: bash
    :class: ignore

    $ symfony run psql < dump.sql

.. warning::

    Ніколи не викликайте ``docker-compose down``, якщо ви не хочете втратити дані. Або спочатку створіть резервну копію.

Додавання PostgreSQL до SymfonyCloud
-----------------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Для продакшн-інфраструктури у SymfonyCloud, додавання такого сервісу, як PostgreSQL, виконується у досі пустому файлі ``.symfony/services.yaml``:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Сервіс ``db`` —  це база даних PostgreSQL (та ж версія, що і для Docker), який ми хочемо розмістити у невеликому контейнері, об'ємом 1 ГБ.

Нам також потрібно "прив’язати" БД до контейнера застосунку, який описаний у ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Сервіс ``db`` типу ``postgresql`` позначається як ``database`` у контейнері застосунку.

Останнім кроком є додавання розширення ``pdo_pgsql`` до середовища виконання PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Ось всі зміни у файлі: ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:8.0

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

Зафіксуйте ці зміни, а потім повторно розгорніть у SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Доступ до бази даних на SymfonyCloud
-------------------------------------------------------

PostgreSQL тепер працює як локально, за допомогою Docker, так і в продакшн, у SymfonyCloud.

Як ми щойно побачили, при виконанні команди ``symfony run psql`` відбувається автоматичне підключення до бази даних, розміщеної у контейнері Docker. Це відбувається завдяки змінним середовища, що встановлені командою ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Якщо ви хочете під'єднатися до PostgreSQL, розміщеному у продакшн контейнері, ви можете відкрити тунель SSH між локальним комп'ютером і інфраструктурою SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

За замовчуванням, змінні середовища сервісів SymfonyCloud не налаштовані на локальному комп'ютері. Ви маєте це зробити явно, використовуючи прапорець ``--expose-env-vars``. Чому? Підключення до бази даних у продакшн — небезпечна операція. Ви можете навести безлад у *реальних* даних. Цей прапорець — це те, чим ви підтверджуєте, що це саме *те*, що ви хочете зробити.

Тепер, підключіться до віддаленої бази даних PostgreSQL, за допомогою команди ``symfony run psql``, як і раніше:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Коли завершите роботу, не забудьте закрити тунель:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Для виконання деяких SQL-запитів до бази даних у продакшн, замість використання інтерактивної оболонки, можна також використовувати команду ``symfony sql``.

Перегляд змінних середовища
----------------------------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose й SymfonyCloud легко працюють із Symfony завдяки змінним середовища.

Перевірте всі змінні середовища, що надані ``symfony``, виконавши ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Змінні середовища ``PG*`` зчитуються утилітою ``psql``. А як щодо інших?

Коли тунель до SymfonyCloud відкрито з прапорцем ``--expose-env-vars``, команда ``var:export`` повертає змінні віддаленого середовища:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

Опис вашої інфраструктури
------------------------------------------------

Можливо, ви ще не усвідомили цього, але наявність інфраструктури, що зберігається у файлах поруч із кодом, дуже допомагає. Docker і Symfony Cloud використовують конфігураційні файли для опису інфраструктури проекту. Зміни у коді й зміни в інфраструктурі є частиною одного і того ж патча, коли нова функція потребує додаткового сервісу.

.. sidebar:: Йдемо далі

    * `Сервіси SymfonyCloud <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `Тунель SymfonyCloud <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Документація по PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `Команди docker-compose <https://docs.docker.com/compose/reference/>`_.
