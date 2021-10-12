Van nul naar productie
======================

Ik hou van snelheid. Ik wil dat ons kleine project zo snel mogelijk live is, in productie. Omdat we nog niets ontwikkeld hebben, zullen we beginnen met het deployen van een mooie en eenvoudige "In opbouw" pagina. Dit wordt leuk!

Zoek even een goeie, ouderwetse geanimeerde GIF op het internet. Hier is `degene <http://clipartmag.com/images/website-under-construction-image-6.gif>`_ die ik ga gebruiken:

.. image:: images/under-construction.gif
    :align: center

Zie je wel dat het leuk gaat worden?

Het project initialiseren
-------------------------

Maak een nieuw Symfony-project met de ``symfony`` CLI tool die we reeds samen geïnstalleerd hebben:

.. code-block:: bash

    $ symfony new guestbook --version=5.2
    $ cd guestbook

Dit commando is een dun laagje bovenop ``Composer`` dat de creatie van Symfony-projecten vergemakkelijkt. Het gebruikt een `project-skelet <https://github.com/symfony/skeleton>`_ dat de minimale dependencies bevat; de Symfony-componenten die nodig zijn voor bijna elk project: een console-tool en een HTTP-abstractie die nodig is om webapplicaties te bouwen.

Als je de GitHub-repository voor de basisstructuur bekijkt zul je merken dat deze bijna leeg is. Alleen een ``composer.json`` bestand. Maar de ``guestbook``-map zit vol met bestanden. Hoe is dat mogelijk? Het antwoord is te vinden in de ``symfony/flex``-package. Symfony Flex is een Composer-plugin die op het installatieproces is ingehaakt. Wanneer een Composer-package een *recipe* heeft, voert Symfony Flex deze uit.

Het startpunt van een Symfony-recipe is het manifest-bestand dat beschrijft welke handelingen uitgevoerd moeten worden om de package automatisch te registreren in een Symfony-applicatie. Je hoeft nooit een README-bestand te lezen om een package met Symfony te installeren. Automatisering is een belangrijke feature van Symfony.

Aangezien Git geïnstalleerd is op onze machine, heeft het ``symfony new`` commando ook een Git-repository voor ons gemaakt en de allereerste commit toegevoegd.

Bekijk de mappenstructuur:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

De ``bin/``-map bevat het belangrijkste CLI startpunt: ``console``. Je gaat dit commando vaak gebruiken.

De ``config/``-map bevat een aantal logische configuratiebestanden. Eén bestand per package. Je zal hier nauwelijks wijzigingen aanbrengen. Het is bijna altijd een goed idee om de standaardconfiguratie aan te houden.

De ``public/``-map is de hoofdmap voor de web omgeving, en het ``index.php``-script is de ingang voor al het dynamische HTTP-verkeer.

De ``src/``-map bevat alle code die je zal gaan schrijven; dat is waar je het grootste deel van jouw tijd zal doorbrengen. Standaard gebruiken alle classes in deze directory de ``App`` PHP namespace. Maar, het is jouw thuis. Jouw code. Jouw domeinlogica. Symfony heeft daar weinig over te zeggen.

De ``var/``-map bevat caches, logs en bestanden die tijdens het gebruik door de applicatie worden gegenereerd. Je kan hier het best van afblijven. Het is de enige map die schrijfbaar moet zijn in productie.

De ``vendor/``-map bevat alle packages die door Composer zijn geïnstalleerd, inclusief Symfony zelf. Dat is ons geheime wapen om productiever te zijn. We willen het wiel niet telkens opnieuw uitvinden. Je zal gebruik maken van bestaande libraries die het harde werk voor jou doen. De map wordt beheerd door Composer. Breng hier nooit wijzigingen in aan.

Dit is voorlopig alles wat je moet weten.

Publiek toegankelijke bestanden aanmaken
----------------------------------------

Alles wat onder ``public/`` staat is toegankelijk via een browser. Als je bijvoorbeeld je geanimeerde GIF-bestand (noem het ``under-construction.gif`` ) naar een nieuwe map ``public/images/`` verplaatst, zal deze beschikbaar zijn op een URL zoals ``https://localhost/images/under-construction.gif``.

Download hier mijn GIF-afbeelding:

.. code-block:: bash

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Een lokale webserver starten
----------------------------

.. index::
    single: Symfony CLI;server:start

De ``symfony``-CLI wordt geleverd met een webserver die geoptimaliseerd is voor ontwikkelwerk. Uiteraard werkt dit prima samen met Symfony, maar het is niet bedoeld om in productie te gebruiken.

Start de webserver in de achtergrond ( ``-d`` parameter), vanuit de projectmap:

.. code-block:: bash

    $ symfony server:start -d

De server gebruikt de eerste beschikbare poort, te beginnen vanaf 8000. Om het makkelijk te maken, opent de CLI de website in een browser op de juiste poort:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Jouw favoriete browser zou op de voorgrond moeten verschijnen en een nieuw tabblad openen dat ongeveer het volgende weergeeft:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    Om problemen op te lossen, voer ``symfony server:log`` uit; het toont continu de logs van de webserver, PHP en jouw applicatie.

Bezoek ``/images/under-construction.gif``. Ziet het er als volgt uit?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

Tevreden? Tijd om ons werk te committen:

.. code-block:: bash
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Een favicon toevoegen
---------------------

Om te voorkomen dat er 404 HTTP-fouten in de logs worden "gespamd" door een ontbrekend favicon, voegen we er nu een toe:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/favicon.ico', 'public/favicon.ico');"
    $ git add public/
    $ git commit -m'Add a favicon'

De productieomgeving voorbereiden
---------------------------------

.. index::
    single: SymfonyCloud;Initialization

Hoe zit het met het deployen van ons werk naar productie? Ik weet het, we hebben nog niet eens een goede HTML-pagina om onze gebruikers te verwelkomen. Maar een klein "under construction"-figuurtje op de productieserver tonen zou een grote stap vooruit zijn. En je kent het motto: *deploy vroegtijdig en frequent* .

Je kunt deze applicatie hosten bij elke provider die PHP ondersteunt... wat betekent dat bijna alle hostingproviders in aanmerking komen. Controleer wel een paar dingen: we willen de laatste PHP-versie gebruiken en de mogelijkheid hebben om een database, een queue en nog een aantal services te hosten.

Mijn keuze is gemaakt, het wordt `SymfonyCloud <https://symfony.com/cloud>`_ . Het biedt alles wat we nodig hebben en bovendien helpt SymfonyCloud met het financieel ondersteunen van de ontwikkeling van Symfony.

.. index::
    single: Symfony CLI;project:init

De ``symfony``-CLI heeft ingebouwde ondersteuning voor SymfonyCloud. Laten we een SymfonyCloud-project starten:

.. code-block:: bash

    $ symfony project:init

Dit commando maakt een aantal bestanden aan die SymfonyCloud nodig heeft, namelijk ``.symfony/services.yaml``, ``.symfony/routes.yaml``, en ``.symfony.cloud.yaml``.

Voeg ze toe aan Git en commit:

.. code-block:: bash

    $ git add .
    $ git commit -m"Add SymfonyCloud configuration"

.. note::

    Het gebruik van het algemene en gevaarlijke ``git add .`` werkt prima als er een ``.gitignore`` bestand is gegenereerd dat automatisch alle bestanden uitsluit die we niet willen committen.

In productie gaan
-----------------

.. index::
    single: Symfony CLI;project:create
    single: Symfony CLI;deploy

Tijd om te deployen?

Maak een nieuw SymfonyCloud-project aan:

.. code-block:: bash

    $ symfony project:create --title="Guestbook" --plan=development

Dit commando doet veel:

* De eerste keer dat je dit commando uitvoert, dien je in te loggen met jouw SymfonyConnect-gegevens, als dat nog niet gebeurd is.

* Het bouwt een nieuw project op SymfonyCloud op (je krijgt 7 dagen *gratis* bij elk nieuw project).

Deploy vervolgens:

.. code-block:: bash

    $ symfony deploy

De code wordt gedeployed door naar de Git-repository te pushen. Aan het einde van het commando heeft het project een specifieke domeinnaam gekregen die je kan gebruiken om toegang te krijgen tot het project.

.. index::
    single: Symfony CLI;open:remote

Controleer of alles goed werkt:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Je zou een 404 moeten krijgen, maar bij het bezoeken van ``/images/under-construction.gif`` krijg je ons werk te zien.

Merk op dat je niet de mooie standaard Symfony-pagina te zien krijgt op SymfonyCloud. Waarom? Je zal erachter komen dat SymfonyCloud meerdere omgevingen ondersteunt en de code automatisch in de productieomgeving geplaatst werd.

.. index::
    single: Symfony CLI;project:delete

.. tip::

    Als je het project op SymfonyCloud wilt verwijderen, gebruikt je het ``project:delete``-commando.

.. sidebar:: Verder gaan

    * De `Symfony Recipes Server <https://flex.symfony.com/>`_ , waar je alle beschikbare recipes voor je Symfony-applicaties kan vinden;

    * De repositories voor de `officiële Symfony-recipes <https://github.com/symfony/recipes>`_ en voor de `recipes van de community <https://github.com/symfony/recipes-contrib>`_ , waar je je eigen recipes toe kan voegen;

    * De `lokale webserver van Symfony <https://symfony.com/doc/current/setup/symfony_server.html>`_;

    * De `SymfonyCloud-documentatie <https://symfony.com/doc/cloud>`_.
