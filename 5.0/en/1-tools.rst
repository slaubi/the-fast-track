Checking your Work Environment
==============================

Before starting to work on the project, we need to check that everyone has a good working environment. It is very important. The developers tools we have at our disposal today are very different from the ones we had 10 years ago. They have evolved a lot, for the better. It would be a shame to not leverage them. Good tools can get you a long way.

Please, don't skip this step. Or at least, read the last section about the Symfony CLI.

A Computer
----------

You need a computer. The good news is that it can run on any popular OS: macOS, Windows, or Linux. Symfony and all the tools we are going to use are compatible with each of these.

Opinionated Choices
-------------------

I want to move fast with the best options out there. I made opinionated choices for this book.

`PostgreSQL`_ is going to be our choice for the database engine.

`RabbitMQ`_ is the winner for queues.

IDE
---

.. index:: IDE

You can use Notepad if you want to. I would not recommend it though.

I used to work with Textmate. Not anymore. The comfort of using a "real" IDE is priceless. Auto-completion, ``use`` statements added and sorted automatically, jumping from one file to another are a few features that will boost your productivity.

I would recommend using `Visual Studio Code`_ or `PhpStorm`_. The former is free, the latter is not but has a better integration with Symfony (thanks to the `Symfony Support Plugin`_). It is up to you. I know you want to know which IDE I am using. I am writing this book in Visual Studio Code.

Terminal
--------

.. index:: Terminal

We will switch from the IDE to the command line all the time. You can use your IDE's built-in terminal, but I prefer to use a real one to have more space.

Linux comes built-in with ``Terminal``. Use `iTerm2`_ on macOS. On Windows, `Hyper`_ works well.

Git
---

.. index:: Git

My last book recommended Subversion for version control. It looks like everybody is using `Git`_ now.

On Windows, install `Git bash`_.

Be sure you know how to do the common operations like running ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``, ...

PHP
---

.. index:: PHP

We will use Docker for services, but I like to have PHP installed on my local computer for performance, stability, and simplicity reasons. Call me old school if you like, but the combination of a local PHP and Docker services is the perfect combo for me.

Use PHP 7.3 if you can, maybe 7.4 depending on when you are reading this book. Check that the :index:`following PHP extensions <PHP extensions>` are installed or install them now: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Optionally install ``redis`` and ``curl`` as well.

You can check the extensions currently enabled via ``php -m``.

We also need ``php-fpm`` if your platform supports it, ``php-cgi`` works as well.

Composer
--------

.. index:: Composer

Managing dependencies is everything nowadays with a Symfony project. Get the latest version of `Composer <https://getcomposer.org/>`_, the package management tool for PHP.

If you are not familiar with Composer, take some time to read about it.

.. tip::

    You don't need to type the full command names: ``composer req`` does the same as ``composer require``, use ``composer rem`` instead of ``composer remove``, ...

Docker and Docker Compose
-------------------------

.. index:: Docker,Docker Compose

Services are going to be managed by Docker and Docker Compose. `Install them <https://docs.docker.com/install/>`_ and start Docker. If you are a first time user, get familiar with the tool. Don't panic though, our usage will be very straightforward. No fancy configurations, no complex setup.

Symfony CLI
-----------

.. index:: Symfony CLI

Last, but not least, we will use the ``symfony`` CLI to boost our productivity. From the local web server it provides, to full Docker integration and SymfonyCloud support, it will be a great time saver.

Install the `Symfony CLI <https://symfony.com/download>`_ and move it under your ``$PATH``. Create a `SymfonyConnect <https://connect.symfony.com/>`_ account if you don't have one already and log in via ``symfony login``.

To use HTTPS locally, we also need to `install a CA <https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls>`_ to enable TLS support. Run the following command:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Check that your computer has all needed requirements by running the following command:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

If you want to get fancy, you can also run the `Symfony proxy <https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy>`_. It is optional but it allows you to get a local domain name ending with ``.wip`` for your project.

When executing a command in a terminal, we will almost always prefix it with ``symfony`` like in ``symfony composer`` instead of just ``composer``, or ``symfony console`` instead of ``./bin/console``.

The main reason is that the Symfony CLI automatically sets some environment variables based on the services running on your machine via Docker. These environment variables are available for HTTP requests because the local web server injects them automatically. So, using ``symfony`` on the CLI ensures that you have the same behavior across the board.

Moreover, the Symfony CLI automatically selects the "best" possible PHP version for the project.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
