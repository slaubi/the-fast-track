Adopter une méthodologie
=========================

Teaching is about repeating the same thing again and again. I won't do that. I
promise. At the end of each step, you should do a little dance and save your
work. It is like ``Ctrl+S`` but for a website.

Mettre en place une stratégie Git
----------------------------------

.. index::
    single: Git;add
    single: Git;commit

À la fin de chaque étape, n'oubliez pas de commiter vos modifications :

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

You can safely add "everything" as Symfony manages a ``.gitignore`` file for
you. And each package can add more configuration. Have a look at the current
content:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,8

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

The funny strings are markers added by Symfony Flex so that it knows what to
remove if you decide to uninstall a dependency. I told you, all the tedious
work is done by Symfony, not you.

It could be nice to push your repository to a server somewhere. GitHub,
GitLab, or Bitbucket are good choices.

If you are deploying on SymfonyCloud, you already have a copy of the Git
repository, but you should not rely on it. It is only for deployment usage. It
is not a backup.

Déploiement continu en production
----------------------------------

.. index::
    single: Symfony CLI;deploy

Another good habit is to deploy frequently. Deploying at the end of each step
is a good pace:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
