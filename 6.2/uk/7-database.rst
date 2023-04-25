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

Для управління сервісами на нашому локальному комп'ютері ми вирішили використовувати Docker. Згенерований файл ``docker-compose.yml`` вже містить PostgreSQL як сервіс:

.. code-block:: yaml
    :caption: docker-compose.yml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-14}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

Це дозволить встановити сервер PostgreSQL і налаштувати деякі змінні середовища, що контролюють ім'я й облікові записи бази даних. Значення, насправді, не є важливими.

Ми також відкриваємо порт PostgreSQL (``5432``) контейнера локальному хосту. Це допоможе отримати доступ до бази даних з нашого комп'ютера:

.. code-block:: yaml
    :caption: docker-compose.override.yml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    Розширення ``pdo_pgsql`` мало бути встановлено разом із PHP, на попередньому кроці.

Запуск Docker Compose
---------------------------

Запустіть Docker Compose у фоновому режимі (``-d``):

.. code-block:: terminal

    $ docker-compose up -d

Зачекайте на запуск бази даних та перевірте, чи все працює правильно:

.. code-block:: terminal
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Якщо контейнери не запустились, або у стовпчику ``State`` не відображається значення ``Up``, перевірте журнали Docker Compose:

.. code-block:: terminal
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

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Якщо у вас немає доступу до бінарного файлу ``psql`` на локальному хості, ви можете запустити його за допомогою ``docker-compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker-compose exec database psql app app

Резервне копіювання і відновлення бази даних
-----------------------------------------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Використовуйте ``pg_dump``, щоб створити резервну копію бази даних:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

І відновіть дані:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Додавання PostgreSQL до Platform.sh
----------------------------------------------

.. index::
    single: Platform.sh;PostgreSQL

Для продакшн-інфраструктури у Platform.sh додавання такого сервісу, як PostgreSQL, виконується у файлі ``.platform/services.yaml``, що вже було зроблено за допомогою рецепту пакету ``webapp``:

.. code-block:: yaml
    :caption: .platform/services.yaml
    :class: ignore

    database:
        type: postgresql:14
        disk: 1024

Сервіс ``database`` є базою даних PostgreSQL (та ж версія, що і для Docker), який ми хочемо розмістити на диску об'ємом 1 ГБ.

Нам також потрібно "зв'язати" БД з контейнером застосунку, який описаний у ``.platform.app.yaml``:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Сервіс ``database`` типу ``postgresql`` позначається як ``database`` у контейнері застосунку.

Переконайтеся, що розширення ``pdo_pgsql`` вже встановлено для середовища виконання PHP:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Доступ до бази даних у Platform.sh
----------------------------------------------------

PostgreSQL тепер працює як локально, за допомогою Docker, так і в продакшн, у Platform.sh.

Як ми щойно побачили, під час виконання команди ``symfony run psql`` відбувається автоматичне підключення до бази даних, розміщеної у контейнері Docker. Це відбувається завдяки змінним середовища, що встановлені командою ``symfony run``.

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Якщо ви хочете під'єднатися до PostgreSQL, розміщеному у продакшн контейнері, ви можете відкрити тунель SSH між локальним комп'ютером і інфраструктурою Platform.sh:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

За замовчуванням сервіси Platform.sh не відображаються як змінні середовища на локальному комп'ютері. Ви маєте налаштувати це явно, виконавши команду ``var:expose-from-tunnel``. Чому? Підключення до до бази даних у продакшн є небезпечною операцією. Ви можете навести безлад у *реальних* даних.

Тепер, підключіться до віддаленої бази даних PostgreSQL, за допомогою команди ``symfony run psql``, як і раніше:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Коли завершите роботу, не забудьте закрити тунель:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Для виконання деяких SQL-запитів до бази даних у продакшн, замість використання інтерактивної оболонки, можна також використовувати команду ``symfony sql``.

Перегляд змінних середовища
----------------------------------------------------

.. index::
    single: Platform.sh;Environment Variables
    single: Symfony CLI;var:export

Docker Compose й Platform.sh легко працюють із Symfony завдяки змінним середовища.

Перевірте всі змінні середовища, що надані ``symfony``, виконавши ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Змінні середовища ``PG*`` зчитуються утилітою ``psql``. А як щодо інших?

Коли тунель до Platform.sh відкрито за допомогою команди ``var:expose-from-tunnel``, команда ``var:export`` повертає змінні віддаленого середовища:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Опис вашої інфраструктури
------------------------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and Platform.sh use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

.. sidebar:: Йдемо далі

    * `Сервіси Platform.sh`_;

    * `Тунель Platform.sh`_;

    * `Документація по PostgreSQL`_;

    * `Команди docker-compose`_.

.. _`Сервіси Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Тунель Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Документація по PostgreSQL`: https://www.postgresql.org/docs/
.. _`Команди docker-compose`: https://docs.docker.com/compose/reference/
