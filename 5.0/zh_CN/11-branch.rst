对代码进行分支
=====================

在项目中有很多方式来组织代码修改的流程。但是直接工作在 Git 的 master 分支上，并且不经测试直接部署到生产环境中，这很可能不是最好的方案。

测试并不仅仅关于单元测试或功能测试，它也应该包括可以用生产数据来检查应用程序的行为。如果你或你周围的 `利益相关者`_ 可以像部署完毕后普通用户那样试用应用程序，这会是一个巨大的优势，它可以让你充满信心地进行部署。当非技术人员也能去确认新功能时，这一点就显得尤为强大。

出于简化和避免重复的考虑，在下面的步骤中，我们仍然会在 Git 的 master 分支上进行所有工作，但是我们先来看看如何使用一个更好的方案。

采用一个 Git 工作流
--------------------------

一个可行的工作流是对每个新功能或问题修复都新建一个分支。这个方案简单有效。

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

创建分支
------------

.. index::
    single: Git;branch
    single: Git;checkout

这个工作流从创建一个 Git 分支开始：

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

这个命令从 ``master`` 分支创建了名为 ``sessions-in-redis`` 的分支。代码和软件环境的配置从此 *分为* 两个分支。

把会话数据（session）存放在 Redis 里
-----------------------------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

你可能从分支名已经猜到了，我们想要把会话数据的存储从文件系统切换为 Redis。

实现这个切换的步骤很典型：

#. 创建一个 Git 分支；

#. 如有需要，更新 Symfony 配置；

#. 如有需要，增加或修改一些代码；

#. 更新 PHP 配置（增加 PHP 的 Redis 扩展）；

#. 更新 Docker 和 SymfonyCloud 的软件环境配置（增加 Redis 服务）；

#. 在本地测试；

#. 在远程测试；

#. 把这个分支合并到 master；

#. 部署到生产环境；

#. 删除这个分支；

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

请你打开本地网站来测试。因为我们没有做页面展示上的改动，而且还没有开始用会话功能，所以一切应该像之前一样正常运行。

部署一个分支
------------------

.. index::
    single: SymfonyCloud;Environment

在部署到生成环境之前，我们必须在和生成环境一样的软件配置中测试这个分支。我们也需要确保在 Symfony 的 ``prod`` 环境中一切正常运行（本地网站则使用了 Symfony 的 ``dev`` 环境）。

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

现在，让我们来根据 *Git 分支* 创建一个 *SymfonyCloud 环境*：

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

这个命令创建了一个新的环境，如下所示：

* 这个分支从当前的 Git 分支（``sessions-in-redis``）集成了代码和软件环境配置；

* 这个分支的数据来自主分支环境（也就是生产环境）所有服务数据的快照，包括文件（比如用户上传的文件）和数据库内容；

* 一个专用的集群会被创建，用来部署代码，数据还有软件环境。

这里的部署和部署到生产环境的步骤是一样的，所以数据库结构迁移也会被执行。这是验证数据库结构迁移是否可以在生产数据上工作的好方法。

这些非 ``master`` 环境和 ``master`` 环境非常相似，除了一些小的差异：比如，默认情况下非 ``master`` 环境不发送邮件。

.. index::
    single: Symfony CLI;open:remote

当部署完成后，用浏览器打开这个新分支的页面：

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

请注意所有的 SymfonyCloud 命令在当前 Git 分支上都可以运行。上面的操作会打开 ``sessions-in-redis`` 分支上部署的 URL，它的形式类似 ``https://sessions-in-redis-xxx.eu.s5y.io/``。

在新环境中测试下网站，你应该会看到在主环境中添加的所有数据。

如果你在 ``master`` 环境下新建更多的会议，它们不会出现在 ``sessions-in-redis`` 环境下，反过来也一样。这些环境是互相独立和隔离的。

如果 master 分支上的代码继续不断更新，你总是可以对 Git 分支进行 rebase，然后部署更新的版本，从而解决代码和软件环境配置的冲突。

.. index::
    single: Symfony CLI;env:sync

你甚至可以把 master 上的数据同步回 ``sessions-in-redis`` 环境：

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

在部署前调试生产部署
------------------------------

.. index::
    single: SymfonyCloud;Debugging

默认情况下，所有 SymfonyCloud 上的环境设置和 ``master``/``prod`` 环境是一样的（也就是 Symfony 的 ``prod`` 环境）。这样你才能在真实条件下测试应用程序。你会觉得像是在生产服务器上直接开发和测试，但此时你却没有直接操作生产服务器的风险。这让我想起来了过去用 FTP 部署时的美好时光。

.. index::
    single: Symfony CLI;env:debug

出问题的时候，你可以切换到 Symfony 的 ``dev`` 环境：

.. code-block:: bash

    $ symfony env:debug

当排查完错误后，返回到生产环境的设置：

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **绝对不要** 在 ``master`` 分支上使用 ``dev`` 环境或 Symfony 分析器；它会让你的应用变得很慢，而且会导致很多严重的安全问题。

在部署前测试生产部署
------------------------------

用生产数据预览即将发布的网站版本，这带来了很多可能性，无论是对于用户界面/前端的回归测试，还是性能测试。`Blackfire <https://blackfire.io>`_ 是用来做性能测试的完美工具。

参考关于“性能”的那个步骤，从而更多了解如何使用 Blackfire 在部署前测试你的代码。

合并到生产环境
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

当你对分支上的改动满意后，把代码和软件环境配置合并到 Git 的 master 分支：

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

然后部署：

.. code-block:: bash

    $ symfony deploy

部署的时候，只有代码和软件环境配置的更新会被推送到 SymfonyCloud；数据不会受到任何影响。

清理
------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

最后，移除 Git 分支和 SymfonyCloud 环境：

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: 深入学习

    * `Git 分支管理 <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_；

    * `Redis docs <https://redis.io/documentation>`_.

.. _`利益相关者`: https://en.wikipedia.org/wiki/Project_stakeholder
