Adoptarea unei metodologii
==========================

Predarea presupune repetarea aceluiași lucru constant. Nu voi face asta. Iți promit. La sfârșitul fiecărui pas, ar trebui să te bucuri puțin și să salvezi modificările. Este ca un ``Ctrl + S``, dar pentru un site web.

Implementarea unei strategii Git
--------------------------------

.. index::
    single: Git;add
    single: Git;commit

La sfârșitul fiecărui pas, nu uita să-ți salvezi modificările în Git:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Poți adăuga „totul” fără griji deoarece Symfony gestionează un fișier ``.gitignore`` pentru tine. Și fiecare pachet poate adăuga linii suplimentare în acest fișier. Aruncă o privire la conținutul curent:

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

Liniile curente cu caractere ciudate sunt markere adăugate de Symfony Flex astfel încât să știe ce să elimine dacă decizi să dezinstalezi o dependență. Ți-am spus, toată munca plictisitoare este făcută de Symfony, nu de tine.

Ar fi frumos să-ți salvezi repozitoriul undeva pe un server. GitHub, GitLab sau Bitbucket sunt niște variante bune.

Dacă lansezi aplicația pe SymfonyCloud, deja ai o copie a repozitoriului Git, dar nu ar trebui să te bazezi pe acesta. Este utilizat doar pentru lansarea noilor versiuni. Nu îl poți folosi pentru găzduirea completă a codului.

Lansarea în producție în mod continuu
----------------------------------------

.. index::
    single: Symfony CLI;deploy

Un alt obicei bun este să lansezi public proiectele tale cât mai frecvent. Lansarea la sfârșitul fiecărui pas definește un ritm bun de lucru:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
