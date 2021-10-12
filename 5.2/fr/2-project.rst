Présentation du projet
=======================

We need to find a project to work on. It is quite a challenge as we need to
find a project large enough to cover Symfony thoroughly, but at the same time,
it should be small enough; I don't want you to get bored implementing similar
features more than once.

Description du projet
---------------------

As the book has to be released during SymfonyCon Amsterdam, it might be nice if
the project is somehow related to Symfony and conferences. What about a
`guestbook <https://en.wikipedia.org/wiki/Guestbook>`_? A livre d'or as we say
in French. I like the old-fashioned and outdated feeling of developing a
guestbook in 2019!

We have it. The project is all about getting feedback on conferences: a list of
conferences on the homepage, a page for each conference, full of nice comments.
A comment is composed of some small text and an optional photo taken during the
conference. I suppose I have just written down all the specifications we need
to get started.

The *project* will contain several *applications*. A traditional web
application with an HTML frontend, an API, and an SPA for mobile phones. How
does that sound?

La maîtrise s’acquiert par la pratique
-----------------------------------------

Learning is doing. Period. Reading a book about Symfony is nice. Coding an
application on your personal computer while reading a book about Symfony is
even better. This book is very special as everything has been done to let you
follow along, code, and be sure to get the same results as I had locally on my
machine when I coded it initially.

The book contains all the code you need to write and all the commands you need
to execute to get the final result. No code is missing. All commands are
written down. This is possible because modern Symfony applications have very
little boilerplate code. Most of the code we will write together is about the
project's *business logic*. Everything else is mostly automated or generated
automatically for us.

À propos du diagramme de l'infrastructure finale
-------------------------------------------------

Even if the project idea seems simple, we are not going to build an "Hello
World"-like project. We won't only use PHP and a database.

The goal is to create a project with some of the complexities you might find in
real-life. Want a proof? Have a look at the final infrastructure of the
project:

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

One of the great benefit of using a framework is the small amount of code
needed to develop such a project:

* 20 PHP classes under ``src/`` for the website;

* 550 PHP Logical Lines of Code (LLOC) as reported by `PHPLOC
  <https://github.com/sebastianbergmann/phploc>`_;

* 40 lines of configuration tweaks in 3 files (via annotations and YAML),
  mainly to configure the backend design;

* 20 lines of development infrastructure configuration (Docker);

* 100 lines of production infrastructure configuration (SymfonyCloud);

* 5 explicit environment variables.

Envie de relever le défi ?

Récupérer le code source du projet
------------------------------------

To continue on the old-fashioned theme, I could have created a CD containing
the source code, right? But what about a Git repository companion instead?

.. index::
    single: Project;Git Repository
    single: Git;clone

Clone the `guestbook repository
<https://github.com/the-fast-track/book-5.0-1>`_ somewhere on your local
machine:

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.0-1 --book guestbook

Ce dépôt contient tout le code source du livre.

Note that we are using ``symfony new`` instead of ``git clone`` as the command
does more than just cloning the repository (hosted on Github under the
``the-fast-track`` organization:
``https://github.com/the-fast-track/book-5.0-1``). It also starts the web
server, the containers, migrates the database, loads fixtures, ... After
running the command, the website should be up and running, ready to be used.

The code is 100% guaranteed to be synchronized with the code in the book (use
the exact repository URL listed above). Trying to manually synchronize changes
from the book with the source code in the repository is almost impossible. I
tried in the past. I failed. It is just impossible. Especially for books like
the ones I write: books that tells you a story about developing a website. As
each chapter depends on the previous ones, a change might have consequences in
all following chapters.

The good news is that the Git repository for this book is *automatically
generated* from the book content. You read that right. I like to automate
everything, so there is a script whose job is to read the book and create the
Git repository. There is a nice side-effect: when updating the book, the script
will fail if the changes are inconsistent or if I forget to update some
instructions. That's BDD, Book Driven Development!

Parcourir le code source
------------------------

Even better, the repository is not just about the final version of the code on
the ``master`` branch. The script executes each action explained in the book
and it commits its work at the end of each section. It also tags each
step and substep to ease browsing the code. Nice, isn't it?

.. index::
    single: Git;checkout

If you are lazy, you can get the state of the code at the end of a step by
checking out the right tag. For instance, if you'd like to read and test the
code at the end of step 10, execute the following:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Like for cloning the repository, we are not using ``git checkout`` but
``symfony book:checkout``. The command ensures that whatever the state you are
currently in, you end up with a functional website for the step you ask for.
**Be warned that all data, code, and containers are removed by this
operation.**

Vous pouvez également récupérer n'importe quelle sous-étape :

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

Again, I highly recommend you code yourself. But if you get stuck, you can
always compare what you have with the content of the book.

.. index::
    single: Git;diff

Vous avez un doute sur l'étape 10.2 ? Récupérez le *diff* :

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Vous voulez savoir quand un fichier a été créé ou modifié ?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

You can also browse diffs, tags, and commits directly on GitHub. This is a
great way to copy/paste code if you are reading a paper book!
