Подготовка базы данных
==========================================

.. index::
    single: Database

Сайт гостевой книги конференции предназначен для сбора отзывов во время конференций. Нам необходимо хранить комментарии, оставленные участниками конференции.

Лучше всего комментарий описывается следующей структурой данных: автор, его электронная почта, текст сообщения и фотография (необязательно). Такого рода информацию лучше всего хранить в традиционной реляционной базе данных.

Мы будем использовать сервер базы данных PostgreSQL.

Добавление PostgreSQL в Docker Compose
-------------------------------------------------

.. index::
    single: Docker;PostgreSQL

Для управления сервисами на локальном компьютере будем использовать Docker. Сгенерированный файл ``docker-compose.yml`` уже содержит определение сервиса для PostgreSQL:

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
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

В результате будет установлен сервер PostgreSQL и настроены переменные окружения для имени базы данных и учётных данных. Конкретные значения сейчас не важны.

Также перенаправим порт PostgreSQL (``5432``) в контейнера на локальный хост, чтобы можно было получить доступ к базе данных с нашего компьютера:

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    Модуль ``pdo_pgsql`` должен был быть установлен на предыдущем шаге, вместе с установкой PHP.

Запуск Docker Compose
---------------------------

Запустите Docker Compose в фоновом режиме (``-d``):

.. code-block:: terminal
    :class: hide

    $ docker-compose down --remove-orphans

.. code-block:: terminal

    $ docker-compose up -d --remove-orphans

Подождите немного, пока база данных запустится, а затем проверьте, что всё работает нормально:

.. code-block:: terminal
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Если работающих контейнеров нет, или в столбце ``State`` не отображается ``Up``, проверьте логи Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker-compose logs

Обращение к локальной базе данных
--------------------------------------------------------------

Using the ``psql`` command-line utility might prove useful from time to time. But you need to remember the credentials and the database name. Less obvious, you also need to know the local port the database runs on the host. Docker chooses a random port so that you can work on more than one project using PostgreSQL at the same time (the local port is part of the output of ``docker-compose ps``).

Если вы запускаете ``psql`` с помощью Symfony CLI, вам не нужно ничего помнить.

Symfony CLI автоматически обнаруживает сервисы Docker, запущенные для проекта, и устанавливает переменные окружения, необходимые ``psql`` для подключения к базе данных.

.. index::
    single: Symfony CLI;run psql

Благодаря этим соглашениям, доступ к базе данных с помощью команды ``symfony run`` становится намного проще:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Если на вашем локальном хосте не установлена команда ``psql``, вы можете запустить её через ``docker-compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker-compose exec database psql app app

Резервное копирование и восстановление базы данных
-----------------------------------------------------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Используйте ``pg_dump``, чтобы выгрузить данные из базы данных:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

Для восстановления данных:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

Добавление PostgreSQL в Platform.sh
----------------------------------------------

.. index::
    single: Platform.sh;PostgreSQL

Добавление такого сервиса, как PostgreSQL, в инфраструктуру продакшена на Platform.sh, делается через изменения в файле ``.platform/services.yaml``, что уже было сделано с помощью рецепта пакета ``webapp``:

.. code-block:: yaml
    :caption: .platform/services.yaml
    :class: ignore

    database:
        type: postgresql:16
        disk: 1024

Сервис ``database`` — это PostgreSQL (такой же версии, что и для Docker), который мы разместим в небольшом контейнере с диском объёмом 1 Гб.

Также необходимо "привязать" БД к контейнеру приложения, который описан в ``.platform.app.yaml``:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Сервис ``database`` типа ``postgresql`` указан как ``database`` в контейнере приложения.

Последним шагом будет добавление PHP-модуля ``pdo_pgsql``:

.. code-block:: yaml
    :caption: .platform.app.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Доступ к базе данных на Platform.sh
------------------------------------------------------

PostgreSQL теперь работает как локально через Docker, так и на продакшене в Platform.sh.

Как мы только что увидели, при запуске ``symfony run psql`` происходит автоматическое подключение к базе данных, размещённой в контейнере Docker. Это происходит благодаря переменным окружения, установленным командой ``symfony run``.

.. index::
    single: Platform.sh;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Если вы хотите подключиться к PostgreSQL, расположенном в контейнерах на продакшене, вы можете открыть SSH-туннель между вашей локальной машиной и инфраструктурой Platform.sh:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

По умолчанию сервисы Platform.sh не отображаются в переменных окружения на локальной машине. Нужно указать это явно, используя флаг ``--expose-env-vars``. Почему? Подключение к базе данных на продакшен-сервере довольно опасно, поскольку есть риск повредить *реальные* данные.

Теперь подключитесь к удалённой базе данных PostgreSQL с помощью команды ``symfony run psql``, как раньше:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Когда завершите работу, не забудьте закрыть туннель:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Для выполнения SQL-запросов к базе данных на продакшене, вместо использования командной оболочки, вы можете использовать команду ``symfony sql``.

Просмотр переменных окружения
--------------------------------------------------------

.. index::
    single: Platform.sh;Environment Variables
    single: Symfony CLI;var:export

Docker Compose и Platform.sh отлично работают с Symfony благодаря переменным окружения.

Просмотрите все переменные окружения, установленные ``symfony``, выполнив ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Переменные окружения, начинающиеся с ``PG*`` используются утилитой ``psql``. А остальные?

Когда туннель к Platform.sh открыт с помощью ``var:expose-from-tunnel``, команда ``var:export`` возвращает переменные из удалённого окружения:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Описание вашей инфраструктуры
--------------------------------------------------------

Возможно, вы ещё этого не осознали, но хранение инфраструктуры в файлах вместе с кодом очень помогает в работе. Настройка инфраструктуры проекта для Docker и Platform.sh производится в конфигурационных файлах. Когда новой функциональности потребуется дополнительный сервис, изменения в коде и инфраструктуре будут зафиксированы в одном патче.

.. sidebar:: Двигаемся дальше

    * `Сервисы Platform.sh`_;

    * `Туннель Platform.sh`_;

    * `Документация PostgreSQL`_;

    * `docker-compose commands`_.

.. _`Сервисы Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Туннель Platform.sh`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`Документация PostgreSQL`: https://www.postgresql.org/docs/
.. _`docker-compose commands`: https://docs.docker.com/compose/reference/
