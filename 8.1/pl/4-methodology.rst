Przyjęcie metodyki
===================

Bardzo często nauka polega na wielokrotnym powtarzaniu tych samych czynności. Obiecuję, że nie będę tego robić. Po zakończeniu każdego etapu zatańcz mały taniec zwycięstwa i zapisz swoją pracę. To takie ``Ctrl+S`` dla strony internetowej.

Wdrażanie strategii Git
------------------------

.. index::
    single: Git;add
    single: Git;commit

Po zakończeniu każdego etapu nie zapomnij o zatwierdzeniu (ang. commit) zmian:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Możesz bezpiecznie dodać "wszystko", ponieważ Symfony zarządza plikiem ``.gitignore``. Co więcej, każdy pakiet może dodać do tego pliku  swoje wykluczenia. Przyjrzyj się bieżącej zawartości:

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

Te zabawne ciągi znaków są znacznikami dodawanymi przez Symfony Flex, który dzięki temu wie, co usunąć, jeśli zdecydujesz się odinstalować zależność. Mówiłem Ci, cała nudna praca jest wykonywana przez Symfony, nie przez Ciebie.

Miło byłoby przenieść repozytorium na serwer Git. Dobrym wyborem będzie GitHub, GitLab czy Bitbucket.

.. note::

    Jeśli korzystasz z Upsun, masz już kopię repozytorium Git, ponieważ wykorzystuje on Git w tle, gdy używasz ``cloud:deploy``. Ale nie należy jej traktować jako kopii zapasowej projektu. Jest wykorzystywana tylko w procesie wdrażania.

Wdrażanie na środowisko produkcyjne w sposób ciągły
--------------------------------------------------------

.. index::
    single: Symfony CLI;cloud:deploy

Innym dobrym nawykiem jest częste wdrażanie. Wdrażanie projektu na końcu każdego etapu to też dobry pomysł:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
