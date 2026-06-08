Adopting a Methodology
======================

Teaching is about repeating the same thing again and again. I won't do that. I promise. At the end of each step, you should do a little dance and save your work. It is like ``Ctrl+S`` but for a website.

Implementing a Git Strategy
---------------------------

.. index::
    single: Git;add
    single: Git;commit

At the end of each step, don't forget to commit your changes:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

You can safely add "everything" as Symfony manages a ``.gitignore`` file for you. And each package can add more configuration. Have a look at the current content:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

The funny strings are markers added by Symfony Flex so that it knows what to remove if you decide to uninstall a dependency. I told you, all the tedious work is done by Symfony, not you.

It could be nice to push your repository to a server somewhere. GitHub, GitLab, or Bitbucket are good choices.

.. note::

    If you are deploying on Platform.sh, you already have a copy of the Git repository as Platform.sh uses Git behind the scenes when you are using ``cloud:push``. But you should not rely on the Platform.sh Git repository. It is only for deployment usage. It is not a backup.

Deploying to Production Continuously
------------------------------------

.. index::
    single: Symfony CLI;cloud:push

Another good habit is to deploy frequently. Deploying at the end of each step is a good pace:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
