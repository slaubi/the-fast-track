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

На нашей локальной машине, для управления сервисами мы решили использовать Docker. Создайте файл ``docker-compose.yaml`` и добавьте PostgreSQL в качестве сервиса:

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

В результате будет установлен сервер PostgreSQL версии 11 и настроены переменные окружения для имени базы данных и учётных данных. Конкретные значения сейчас не важны.

Также откроем порт PostgreSQL (``5432``) у контейнера для доступа к нему с локального хоста, чтобы получить доступ к базе данных с нашего компьютера.

.. note::

    Модуль ``pdo_pgsql`` должен был быть установлен на предыдущем шаге, вместе с установкой PHP.

Запуск Docker Compose
---------------------------

Запустите Docker Compose в фоновом режиме (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Подождите немного, пока база данных запустится, а затем проверьте, что всё работает нормально:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Если работающих контейнеров нет, или в столбце ``State`` не отображается ``Up``, проверьте логи Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Обращение к локальной базе данных
--------------------------------------------------------------

Использование консольной программы ``psql`` может пригодится в отдельных случаях. Хотя для этого вам нужно помнить учётные данные и имя базы. Вам также нужно знать локальный порт, на котором запущена база данных. Docker выбирает произвольный порт для того, чтобы вы могли одновременно работать над разными проектами, использующими PostgreSQL (локальный порт отображается в выводе команды ``docker-compose ps``).

Если вы запускаете ``psql`` с помощью Symfony CLI, вам не нужно ничего помнить.

Symfony CLI автоматически обнаруживает сервисы Docker, запущенные для проекта, и устанавливает переменные окружения, необходимые ``psql`` для подключения к базе данных.

.. index::
    single: Symfony CLI;run psql

Благодаря этим соглашениям, доступ к базе данных с помощью команды ``symfony run`` становится намного проще:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Добавление PostgreSQL в SymfonyCloud
-----------------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Добавление такого сервиса, как PostgreSQL, в инфраструктуру продакшена на SymfonyCloud, делается через изменения в файле ``.symfony/services.yaml``, который пока ещё пуст:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

Сервис ``db`` — это PostgreSQL 11 (как и в Docker), который мы разместим в небольшом контейнере с диском объёмом 1 Гб.

Также необходимо "привязать" БД к контейнеру приложения, который описан в ``.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

Сервис ``db`` типа ``postgresql`` указан как ``database`` в контейнере приложения.

Последним шагом будет добавление PHP-модуля ``pdo_pgsql``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Вот полный результат изменений в файле ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

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

Зафиксируйте изменения в репозитории, а затем повторно разверните в SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Доступ к базе данных на SymfonyCloud
-------------------------------------------------------

PostgreSQL теперь работает как локально через Docker, так и на продакешене в SymfonyCloud.

Как мы только что увидели, при запуске ``symfony run psql`` происходит автоматическое подключение к базе данных, размещённой в контейнере Docker. Это происходит благодаря переменным окружения, установленным командой ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Если вы хотите подключиться к PostgreSQL, расположенном в контейнерах на продакшене, вы можете открыть SSH-туннель между вашей локальной машиной и инфраструктурой SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

По умолчанию сервисы SymfonyCloud не отображаются в переменных окружения на локальной машине. Вы должны сделать это явно, используя флаг ``--expose-env-vars``. Почему? Подключение к базе данных на продакшене является опасной операцией. Вы можете испортить *реальные* данные. Указывая этот флаг, вы подтверждаете, что *действительно* хотите это сделать.

Теперь подключитесь к удалённой базе данных PostgreSQL с помощью команды ``symfony run psql``, как раньше:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Когда завершите работу, не забудьте закрыть туннель:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Для выполнения SQL-запросов к базе данных на продакшене, вместо использования командной оболочки, вы можете использовать команду ``symfony sql``.

Просмотр переменных окружения
--------------------------------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

Docker Compose и SymfonyCloud отлично работают с Symfony благодаря переменным окружения.

Просмотрите все переменные окружения, установленные ``symfony``, выполнив ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

Переменные окружения, начинающиеся с ``PG*`` используются утилитой ``psql``. А остальные?

Когда туннель к SymfonyCloud открыт с установленным флагом ``--expose-env-vars``, команда ``var:export`` возвращает переменные из удалённого окружения:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: Двигаемся дальше

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Документация PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `Команды docker-compose. <https://docs.docker.com/compose/reference/>`_.
