排查问题
============

拥有正确的工具对问题进行调试，这也是设置项目的一部分。

安装更多的依赖包
------------------------

你回想一下，这个项目创建时只安装了很少的依赖包。没有模板引擎，没有调试工具，没有日志系统。我们的想法是，你在需要时能安装更多依赖。如果只是开发一个 HTTP 的 API 或者一个命令行工具，那何必要依赖模板引擎呢？

如何添加更多依赖包呢？借助 Composer。除了“常规”的 Composer 包，我们也会用到两种“特殊”的包。

* *Symfony 组件*：这些包实现了一些核心功能，以及大部分应用程序都需要的一些低层抽象（路由、控制台、HTTP 客户端、Mailer、缓存......）

* *Symfony Bundles*：这些包要么是增加了高层的功能，要么是提供了和第三方库的整合（bundle 主要由社区贡献）。

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

作为开始，让我们来添加 Symfony 分析器（profiler）的包。当你需要去找到程序的问题根源时，它可以为你节省大量时间。

.. code-block:: terminal

    $ symfony composer req profiler --dev

``profiler`` 是 ``symfony/profiler-pack`` 包的一个别名。

*别名* 不是 Composer 的功能，它是 Symfony 提供的一个概念，可以让你工作起来更方便。别名是一些流行 Composer 包的快捷方式。想要为你的应用找一个 ORM？用 ``orm`` 别名。想要开发一个 API？用 ``api`` 别名。这些别名会被自动解析成一个或多个常规 Composer 包。解析的规则是由 Symfony 核心团队基于自己的意见制定的。

还有一个很巧妙的功能，就是在包的名字里，你总是可以省略 ``symfony``。比如你可以用 ``cache`` 代替 ``symfony/cache``。

.. tip::

    你还记得我们之前提过一个叫 ``symfony/flex`` 的 Composer 插件吗？别名就是它的功能之一。

理解 Symfony 环境
---------------------

.. index::
    single: Symfony Environments

你有没有注意到，当我们执行 ``composer req`` 命令时，用了 ``--dev`` 选项？因为 Symfony 分析器只是在开发时有用，所以我们要避免把它装在生产环境中。

Symfony 支持 *环境* 这个概念。默认情况下，它内置支持 3 种环境：``dev``、``prod`` 和 ``test``，但你也能添加任意多的环境。所有的环境共享同一套代码，但它们代表了不同的 *配置*。

举个例子，在 ``dev`` 环境中，所有的调试工具会启用，而在 ``prod`` 环境中，应用的性能会被优化。

改变 ``APP_ENV`` 这个环境变量，就可以从一个环境切换到另一个。

当你部署到 Upsun 上时，环境会被自动切换到 ``prod`` （这个值存储在 ``APP_ENV`` 里）。

管理环境配置
------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` 可以在终端里用“真正”的环境变量来设置：

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

在生产服务器上，用真正的环境变量来设置类似 ``APP_ENV`` 这样的值是推荐的做法。但在开发机器上，定义很多环境变量会有点繁琐，所以我们在一个 ``.env`` 文件中定义它们。

当新建项目时，一个带有合理初始值的 ``.env`` 文件会自动生成。

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    借助 Symfony Flex 使用的 recipe，任何包都可以添加新的环境变量到这个文件中。

``.env`` 文件会被提交到代码仓库，它的内容是生产环境下的 *默认* 值。你可以通过新建 ``.env.local`` 文件来覆盖那些默认值。这个 ``.env.local`` 文件不应该被提交到 Git，这也是为什么 ``.gitignore`` 文件已经将它忽略了。

绝不要把机密或敏感的数据存储在这些文件中。我们会在后面看到如何管理机密内容。

用日志记录所有发生的事
---------------------------------

.. index::
    single: Logger

新项目自带的日志和调试功能很有限。我们来装一些新工具以获得帮助，使我们无论在开发环境还是生产环境里，都可以更好地排查问题。

.. code-block:: terminal

    $ symfony composer req logger

.. index::
    single: Components;Debug
    single: Debug

我们只在开发环境中安装调试工具：

.. code-block:: terminal

    $ symfony composer req debug --dev

学习 Symfony 的调试工具
------------------------------

刷新一下首页，现在你会看到窗口底部多了一个工具栏。

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

可能你第一个注意到的就是红色的 **404**。记住这个页面是一个占位页，因为我们还没有开发首页。尽管这个缺省的欢迎页很漂亮，它仍然是一个错误页面，所以当前的 HTTP 状态码是 404，而不是 200。多亏了这个 *web 调试工具栏*，你一下子就得到了这些信息。

如果你点击那个小的感叹号，你会看到“真正”的异常信息，这些信息也是 Symfony 分析器日志的一部分。如果你要查看堆栈跟踪，点击左侧菜单的“Exception”链接。

无论什么时候你的代码出了问题，你总是能看到类似下图的异常页面，它提供了你所需的一切信息，来让你明白这个问题以及它的源头。

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

你可以在这个工具栏里到处点击，花点时间探索一下 Symfony 分析器提供的各种信息。

.. index::
    single: Symfony CLI;server:log

日志对于调试也很有用。Symfony 有一个很方便的命令可以查看最新的日志（来自 web 服务器、PHP 和应用程序的日志）：

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

我们来做个小实验。打开 ``public/index.php``，破坏掉里面的PHP代码（比如在代码中间加个 foobar）。在浏览器中刷新页面，然后观察日志输出流：

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

输出的内容被很美观地高亮了，这样你就可以注意到错误信息。

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

另一个很棒的调试助手是 Symfony 的 ``dump()`` 函数。它始终可用，你可以用它来导出复杂的变量，变量会以美观且可交互式的格式展现出来。

临时更改下 ``public/index.php`` ，让它导出 Request 对象：

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -18,5 +18,8 @@ if ($_SERVER['APP_DEBUG']) {
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);
    +
    +dump($request);
    +
     $response->send();
     $kernel->terminate($request, $response);

刷新页面后，注意下工具栏里的出现了新的“目标”图标，它可以让你检查导出的变量。点击那个图标会打开一个另外的页面，在那里查看导出的变量会更方便。

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

在提交该步骤里的其它更改到仓库前，先撤销在这里的更改。

.. code-block:: terminal

    $ git checkout public/index.php

配置你的 IDE
----------------

开发环境下有异常抛出时，Symfony 会显示一个带有异常信息和堆栈跟踪的页面。当显示文件路径时，它会自动增加一个链接，这个链接可以在你首选的 IDE 中打开该文件，并定位到正确的行。为了使用这个功能，你需要去配置你的 IDE。Symfony 自带支持很多 IDE；我在这个项目里使用的是 Visual Studio Code：

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -6,3 +6,4 @@ max_execution_time=30
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

链接的文件不仅限于异常。例如在配置了 IDE 后，*web 调试工具栏* 里的控制器也会变成可点击的。

在生产环境中排错
------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;logs
    single: Symfony CLI;ssh

在生产服务器上调试总是更为棘手。比如，你无法使用 Symfony 分析器。日志提供的信息也更少。但是显示最近的日志是可以的：

.. code-block:: terminal
    :class: ignore

    $ symfony logs

你甚至可以通过 SSH 连接到 web 容器。

.. code-block:: terminal
    :class: ignore

    $ symfony ssh

别担心，你不太会把东西弄坏。大部分文件都是只读的。在生产环境中你无法进行热修复，但你在本书后面会学到一个好得多的方法。

.. sidebar:: 深入学习

    * `SymfonyCasts：环境和配置文件教程 <https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files>`_；

    * `SymfonyCasts ：环境变量教程 <https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables>`_；

    * `SymfonyCasts 的 web 调试工具栏和分析器教程 <https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler>`_；

    * 在 Symfony 应用中 `管理多个 .env 文件 <https://symfony.com/doc/current/configuration.html#managing-multiple-env-files>`_。
