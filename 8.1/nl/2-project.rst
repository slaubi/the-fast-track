Inleiding van het project
=========================

We hebben een project nodig om aan te werken. Dat is een hele uitdaging, omdat we een project moeten vinden dat groot genoeg is om Symfony grondig te behandelen, maar tegelijkertijd klein genoeg is dat je je niet verveelt door telkens dezelfde functionaliteit te moeten implementeren.

Onthulling van het project
--------------------------

Het zou leuk zijn als het project op de een of andere manier gerelateerd is aan Symfony en de community. Wat denk je van een `gastenboek`_, aangezien we elk jaar vrij veel online en fysieke conferenties organiseren? Een "livre d'or" zoals we in het Frans zeggen. Ik hou van het ouderwetse en verouderde gevoel van het ontwikkelen van een gastenboek in de 21ste eeuw!

We hebben het. Het project draait om het krijgen van feedback op conferenties: een lijst met conferenties op de homepage, een pagina voor elke conferentie, vol met leuke opmerkingen. Een opmerking bestaat uit een kleine tekst en een optionele foto die tijdens de conferentie is gemaakt. Ik geloof dat ik net alle specificaties heb opgeschreven die we nodig hebben om te beginnen.

Het *project* zal verschillende *toepassingen* bevatten. Een traditionele webapplicatie met een HTML-frontend, een API en een SPA voor mobiele telefoons. Klinkt goed toch?

Leren is doen
-------------

Leren is doen. Punt uit. Het lezen van een boek over Symfony is leuk. Het bouwen van een applicatie op jouw computer tijdens het lezen van een boek over Symfony is nog beter. Dit boek is heel bijzonder, omdat er alles aan is gedaan om je mee te laten doen, te coderen en er zeker van te zijn dat je dezelfde resultaten krijgt als ik lokaal op mijn machine had toen ik het in eerste instantie codeerde.

Het boek bevat alle code die je dient te schrijven en alle commando's die uitgevoerd moeten worden om het eindresultaat te krijgen. Er ontbreekt geen code. Alle commando's zijn opgeschreven. Dit is mogelijk omdat moderne Symfony-toepassingen zeer weinig "boilerplate code" hebben. Het grootste deel van de code die we samen zullen schrijven gaat over de *bedrijfslogica* van het project. Al het andere is grotendeels geautomatiseerd of wordt automatisch voor ons gegenereerd.

Een blik op het uiteindelijke infrastructuurdiagram
---------------------------------------------------

Ook al lijkt het projectidee eenvoudig, we gaan geen "Hello World"-achtig project bouwen. We gebruiken niet alleen PHP en een database.

Het doel is om een project te creëren met een aantal van de complexiteiten die je in het echte leven zou kunnen vinden. Wil je bewijs? Bekijk de uiteindelijke infrastructuur van het project:

.. figure:: images/infrastructure.png
    :align: center
    :figclass: ad diagram

Eén van de grote voordelen van het gebruik van een framework is de kleine hoeveelheid code die nodig is om een dergelijk project te ontwikkelen:

* 20 PHP-classes onder ``src/`` voor de website;

* 550 PHP Logical Lines of Code (LLOC) zoals gerapporteerd door `PHPLOC`_ ;

* 40 regels zijn aangepast in 3 configuratiebestanden (via attributen en YAML), voornamelijk om de backend te configureren;

* 20 regels voor de configuratie van de ontwikkelingsinfrastructuur (Docker);

* 100 regels voor de configuratie van de productie-infrastructuur (Upsun);

* 5 expliciete omgevingsvariabelen.

Klaar voor de uitdaging?

De broncode van het project verkrijgen
--------------------------------------

Om verder te gaan met het ouderwetse thema had ik een CD kunnen maken met de broncode, toch? Maar in plaats daarvan, wat dacht je van een Git-repository?

.. index::
    single: Project;Git Repository
    single: Git;clone

Clone het `gastenboek repository`_ ergens op jouw lokale machine:

.. code-block:: terminal
    :class: ignore

    $ symfony new --version=8.1-1 --book guestbook

Deze repository bevat alle code van het boek.

Let wel dat we ``symfony new`` gebruiken in plaats van ``git clone``, omdat dat laatste commando niets meer doet dan alleen het klonen van de repository (gehost op Github onder de ``the-fast-track-organisatie``: ``https://github.com/the-fast-track/book-6.2-1`` ). Het start ook de webserver, de containers, migreert de database, laadt de fixtures, ... Na het uitvoeren van het commando zou de website operationeel moeten zijn, klaar voor gebruik.

De code is 100% gegarandeerd synchroon met de code in het boek (gebruik de exacte repository-URL zoals hierboven vermeld). Proberen om handmatig wijzigingen uit het boek te synchroniseren met de broncode in de repository is bijna onmogelijk. Ik heb het in het verleden geprobeerd. Ik heb gefaald. Het is simpelweg onmogelijk. Zeker voor het soort boeken dat ik schrijf: boeken die een verhaal vertellen over het ontwikkelen van een website. Aangezien elk hoofdstuk afhankelijk is van de vorige, kan een wijziging gevolgen hebben voor alle volgende hoofdstukken.

Het goede nieuws is dat de Git-repository voor dit boek *automatisch gegenereerd* wordt op basis van de inhoud van het boek. Dat heb je goed gelezen. Ik hou ervan om alles te automatiseren, dus er is een script dat als taak heeft om het boek te lezen en de Git repository te maken. Dat geeft een leuk neveneffect: bij het bijwerken van het boek zal het script falen als de wijzigingen niet consistent zijn, of als ik vergeet een aantal instructies bij te werken. Dat noemen we BDD, Book Driven Development!

Navigeren door de broncode
--------------------------

Beter nog, de repository gaat niet alleen over de definitieve versie van de code op de ``main`` branch. Het script voert elke actie uit die in het boek wordt uitgelegd en het commit zijn werk aan het einde van elke sectie. Het plaatst ook een tag bij elke stap en substap om het browsen door de code te vergemakkelijken. Mooi toch?

.. index::
    single: Git;checkout

Als je lui bent, kun je de staat van de code aan het einde van een stap zien door naar de juiste tag te gaan. Als je bijvoorbeeld de code aan het einde van stap 10 wilt lezen en testen, voer dan het volgende uit:

.. code-block:: terminal
    :class: ignore

    $ symfony book:checkout 10

Net als bij het klonen van de repository, gebruiken we niet ``git checkout``, maar ``symfony book:checkout``. Het commando zorgt ervoor dat je, in welke staat je je ook bevindt, een functionele website krijgt voor de stap waar je om vraagt. **Wees gewaarschuwd dat alle gegevens, code en containers door deze actie worden verwijderd.**

Je kunt ook elke substap uitchecken:

.. code-block:: terminal
    :class: ignore

    $ symfony book:checkout 10.2

Nogmaals, ik raad je ten zeerste aan om zelf te coderen. Maar als je vast komt te zitten, kun je jouw code altijd vergelijken met de inhoud van het boek.

.. index::
    single: Git;diff

Weet je niet zeker of je alles goed hebt gedaan in substap 10.2? Zo zie je de verschillen:

.. code-block:: terminal
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Wil je weten wanneer een bestand is aangemaakt of gewijzigd?

.. code-block:: terminal
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

Je kunt ook direct op GitHub door diffs, tags en commits bladeren. Dit is een geweldige manier om code te kopiëren/plakken als je een papieren boek leest!

.. _`PHPLOC`: https://github.com/sebastianbergmann/phploc
.. _`gastenboek`: https://en.wikipedia.org/wiki/Guestbook
.. _`gastenboek repository`: https://github.com/the-fast-track/book-5.0-1
