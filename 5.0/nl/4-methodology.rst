Een methodiek eigen maken
=========================

Bij het aanleren van iets nieuws val je vaak in herhaling. Dat zal ik niet doen, beloofd! Aan het einde van elke stap moet je wel een klein dansje doen en je wijzigingen opslaan. Het is als ``Ctrl+S``, maar dan voor een website.

Een Git-strategie implementeren
-------------------------------

.. index::
    single: Git;add
    single: Git;commit

Vergeet niet om aan het einde van elke stap je werk te committen:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Je kunt veilig "alles" toevoegen omdat Symfony een ``.gitignore`` gebruikt. Bovendien kan elke package meerdere configuraties toevoegen. Neem een kijkje in de huidige inhoud:

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

De rare markeringen worden door Symfony Flex toegevoegd, zodat bij het verwijderen van een package de juiste configuratie mee opgeruimd kan worden. Zoals ik al zei, het vervelende werk wordt door Symfony gedaan, niet door jou.

Het is een goed idee om je repository ergens naar een server te pushen. GitHub, GitLab of Bitbucket zijn uitstekend hiervoor.

Als je gebruik maakt van SymfonyCloud heb je al een kopie van de Git-repository. Vertrouw er niet op dat dit je backup is en gebruik hem enkel voor deployments.

Continu deployen naar productie
-------------------------------

.. index::
    single: Symfony CLI;deploy

Een andere goede gewoonte is om regelmatig te deployen, bijvoorbeeld aan het einde van elke stap:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
