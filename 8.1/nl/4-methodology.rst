Een methodiek eigen maken
=========================

Bij het aanleren van iets nieuws val je vaak in herhaling. Dat zal ik niet doen, beloofd! Aan het einde van elke stap moet je wel een klein dansje doen en je wijzigingen opslaan. Het is als ``Ctrl+S``, maar dan voor een website.

Een Git-strategie implementeren
-------------------------------

.. index::
    single: Git;add
    single: Git;commit

Vergeet niet om aan het einde van elke stap je werk te committen:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Je kunt veilig "alles" toevoegen omdat Symfony een ``.gitignore`` gebruikt. Bovendien kan elke package meerdere configuraties toevoegen. Neem een kijkje in de huidige inhoud:

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

De rare markeringen worden door Symfony Flex toegevoegd, zodat bij het verwijderen van een package de juiste configuratie mee opgeruimd kan worden. Zoals ik al zei, het vervelende werk wordt door Symfony gedaan, niet door jou.

Het is een goed idee om je repository ergens naar een server te pushen. GitHub, GitLab of Bitbucket zijn uitstekend hiervoor.

.. note::

    Als je gebruik maakt van Platform.sh heb je al een kopie van de Git-repository omdat Platform.sh achter de schermen gebuik maakt van Git wanneer je ``cloud:deploy`` gebruikt. Maar je moet niet op de Platform.sh repository vertrouwen. Deze dient enkel voor deployments en moet niet beschouwd worden als backup.

Continu deployen naar productie
-------------------------------

.. index::
    single: Symfony CLI;cloud:deploy

Een andere goede gewoonte is om regelmatig te deployen, bijvoorbeeld aan het einde van elke stap:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:deploy
