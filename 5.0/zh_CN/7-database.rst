设置数据库
===============

.. index::
    single: Database

这个会议留言本网站要收集关于会议的反馈。我们要把参会人员贡献的评论保存在一个持久存储中。

如下这个固定的数据结构可以很好地描述一条评论：留言者、电子邮箱、反馈的文字和一张可选的照片。这种类型的数据最适合存放在传统的关系型数据库中。

PostgreSQL 是我们选用的数据库。

把 PostgreSQL 添加到 Docker Compose
---------------------------------------

.. index::
    single: Docker;PostgreSQL

在我们的本地电脑上，我们决定用Docker来管理服务。创建一个 ``docker-compose.yaml`` 文件，加入 PostgreSQL 作为一个服务：

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

这个步骤会安装第 11 版的 PostgreSQL 服务器，并配置一些环境变量来设置数据库名和账号密码。具体的值不重要。

我们把容器的 PostgreSQL 端口（``5432``）暴露给本地机器，这样它就可以连接到数据库了。

.. note::

    在前面安装 PHP 的步骤里，``pdo_pgsql`` 扩展应该已经装好了。

启动 Docker Compose
---------------------

在后台运行 Docker Compose（使用 ``-d`` 选项）：

.. code-block:: bash

    $ docker-compose up -d

先稍等一下，让数据库启动，然后检查是否一切正常：

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

如果没有容器在运行，或者 ``State`` 栏不是显示为 ``Up``，检查一下Docker Compose 的日志：

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

连入本地数据库
---------------------

你时不时会用到 ``psql`` 这个命令行小工具。但是你需要记住数据库名以及账号密码。更加不太为人所熟悉的一点是，你还需要知道数据库运行时的本地端口。Docker 会选用一个随机端口，这样你就可以同时开发多个使用了 PostgreSQL 的项目（本地端口是 ``docker-compose ps`` 命令输出中的一部分）。

如果你在 ``symfony`` 命令里使用 ``psql``，你就不必记住任何信息。

``symfony`` 命令会自动检测到项目运行的 Docker 服务，它会把 ``psql`` 连接数据库所需的环境变量暴露出来。

.. index::
    single: Symfony CLI;run psql

多亏这些约定，使用 ``symfony run`` 来连接数据库方便多了。

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

在 SymfonyCloud 中加入 PostgreSQL
-------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

在 SymfonyCloud 的生产环境软件设施里，添加一个诸如 PostgreSQL 这样的服务，是通过修改 ``.symfony/services.yaml`` 这个文件来完成的，该文件目前还是空的：

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

这个 ``db`` 服务是一个第 11 版的 PostgreSQL 数据库（和 Docker 里一样），我们会让它在一个 1GB 磁盘空间的小容器中运行。

我们也需要把数据库“链接”到应用的容器，这需要去编辑 ``.symfony.cloud.yaml`` 文件：

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

在应用容器中，``postgresql`` 类型的 ``db`` 服务被称为 ``database``。

最后一步是在 PHP 运行时中添加 ``pdo_pgsql`` 扩展。

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

这里是对 ``.symfony.cloud.yaml`` 文件改动的全部差别比对：

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

提交这些改变，然后重新部署到 SymfonyCloud：

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

连入 SymfonyCloud 数据库
-----------------------------

现在 PostgreSQL 在本地借助 Docker 运行，同时也在 SymfonyCloud 的生产环境中运行。

正如我们所见，运行 ``symfony run psql`` 会自动连接到 Docker 里的数据库，这要归功于 ``symfony run`` 暴露出的环境变量。

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

如果你想要连接到生产环境容器中的 PostgreSQL，你需要打开一个 SSH 隧道来接通你的本地电脑和 SymfonyCloud 平台设施：

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

默认情况下，SymfonyCloud 服务不会在你的本地电脑上以环境变量暴露出来。要想把它们暴露出来，你必须明确设置 ``--expose-env-vars`` 选项。为什么要这样呢？连接到生产数据库是一个危险的操作。你可能会破坏 *真实的* 数据。强制设置那个选项，是要你确认这 *的确是* 你想要做的操作。

现在，用 ``symfony run psql`` 连接到远程的 PostgreSQL 服务器，和以前一样：

.. code-block:: bash
    :class: ignore

    $ symfony run psql

连接完后，不要忘了关掉隧道：

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    如果要在生产数据库上直接运行一些 SQL 语句，而不是获得一个 shell，你可以用 ``symfony sql`` 命令。

暴露环境变量
------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

借助于环境变量，Docker Compose 和 SymfonyCloud 可以和 Symfony 无缝对接。

执行 ``symfony var:export`` 可以查看 ``symfony`` 暴露出的所有环境变量：

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

``psql`` 小工具会读取 ``PG*`` 形式的环境变量。那其它环境变量呢？

如果一条隧道设置了 ``--expose-env-vars`` 选项并且连接到了 SymfonyCloud，那么 ``var:export`` 命令就会返回远程的环境变量：

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: 深入学习

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `PostgreSQL 文档 <https://www.postgresql.org/docs/>`_；

    * `docker-compose 命令 <https://docs.docker.com/compose/reference/>`_。
