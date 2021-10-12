Het ontdekken van Symfony Internals
===================================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

We gebruiken Symfony al geruime tijd om een krachtige applicatie te ontwikkelen, maar het grootste deel van de code die door de applicatie wordt uitgevoerd komt van Symfony. Een paar honderd regels code tegenover duizenden regels code.

Ik wil graag begrijpen hoe dingen achter de schermen werken. En ik ben altijd gefascineerd geweest door tools die me helpen begrijpen hoe dingen werken. De eerste keer dat ik een stapsgewijze debugger gebruikte of de eerste keer dat ik ``ptrace`` ontdekte zullen magische herinneringen blijven.

Wil je beter begrijpen hoe Symfony werkt? Tijd om onder de motorkap te kijken en te ontdekken hoe Symfony jouw applicatie laat werken. In plaats van te beschrijven hoe Symfony een HTTP-verzoek vanuit een theoretisch perspectief behandelt, wat nogal saai zou zijn, gaan we Blackfire gebruiken om wat visuele representaties te krijgen en enkele geavanceerde onderwerpen aan te snijden.

Symfony internals begrijpen met Blackfire
-----------------------------------------

Je weet al dat alle HTTP-verzoeken worden bediend door één enkel toegangspunt: het ``public/index.php``-bestand. Maar wat gebeurt er daarna? Hoe worden controllers aangeroepen?

Laten we de Engelse homepage in productie met Blackfire profileren via de Blackfire browserextensie:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

Of rechtstreeks via de commandline:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Ga naar de "Tijdlijn"-weergave van het profiel, je ziet dan iets als het volgende:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

Hover op de tijdlijn over de gekleurde balken voor meer informatie over iedere call; je leert zo meer over hoe Symfony werkt:

* Het belangrijkste startpunt is ``public/index.php``;

* De ``Kernel::handle()`` methode verwerkt het verzoek;

* Het roept de ``HttpKernel`` aan, die enkele events uitstuurt;

* Het eerste event is het ``RequestEvent``;

* De ``ControllerResolver::getController()`` methode wordt aangeroepen om te bepalen welke controller moet worden opgeroepen voor de inkomende URL;

* De ``ControllerResolver::getArguments()`` methode wordt aangeroepen om te bepalen welke argumenten aan de controller moeten worden doorgegeven (de param-converter wordt aangeroepen);

* De ``ConferenceController::index()`` methode wordt aangeroepen en de meeste van onze code wordt daar uitgevoerd;

* De ``ConferenceRepository::findAll()`` methode haalt alle conferenties op uit de database (let op de verbinding met de database via ``PDO::__construct()`` );

* De ``Twig\Environment::render()`` methode rendert de template;

* De ``ResponseEvent`` en de ``FinishRequestEvent`` events worden uitgestuurd, maar het lijkt er op dat er geen listeners zijn geregistreerd aangezien ze erg snel uitgevoerd worden.

De tijdlijn is een handige manier om te begrijpen hoe code in elkaar zit; wat van pas kan komen als je op een project moet werken dat door iemand anders ontwikkeld werd.

Profileer nu dezelfde pagina van de lokale machine in de ontwikkelomgeving:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Open het profiel. Je zou doorgestuurd moeten worden naar de call graph weergave, omdat het verzoek heel snel was en de tijdlijn vrij leeg zou zijn:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Begrijp je wat er aan de hand is? De HTTP-cache is ingeschakeld en daardoor profileren we de Symfony HTTP cache laag. Aangezien de pagina in de cache zit, zal ``HttpCache\Store::restoreResponse()`` de HTTP-response uit de cache nemen en wordt de controller nooit aangeroepen.

Schakel de cache-laag uit in ``public/index.php`` zoals in de vorige stap en probeer het opnieuw. Je kan meteen zien dat het profiel er heel anders uitziet:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

De belangrijkste verschillen zijn de volgende:

* Het ``TerminateEvent``, dat niet zichtbaar was in productie, neemt een groot percentage van de uitvoeringstijd in beslag; als je beter kijkt, zie je dat dit het event is dat verantwoordelijk is voor het opslaan van de gegevens van de Symfony profiler die tijdens de request werden verzameld;

* Onder de ``ConferenceController::index()`` wordt de method ``SubRequestHandler::handle()`` uitgevoerd die de ESI rendered (daarom hebben we twee calls naar ``Profiler::saveProfile()``, één voor het hoofdrequest en één voor de ESI).

Verken de tijdlijn om meer te weten te komen; schakel over naar de call graph weergave om een andere beeld te krijgen van dezelfde gegevens.

Zoals we zojuist hebben ontdekt is de code die tijdens ontwikkeling en productie wordt uitgevoerd heel anders. De ontwikkelomgeving is langzamer omdat de Symfony profiler veel gegevens probeert te verzamelen om het debuggen te vergemakkelijken. Daarom moet je altijd profileren in de productieomgeving, ook lokaal.

Enkele interessante experimenten: profileer een foutpagina, profileer de ``/`` pagina (dat is een redirect), of een API resource. Elk profiel vertelt je iets meer over hoe Symfony werkt, welke classes/methods worden aangeroepen, welke operaties zwaar doorwegen en welke licht zijn.

De Blackfire debug addon gebruiken
----------------------------------

.. index::
    single: Blackfire;Debug Addon

Standaard verwijdert Blackfire alle method calls die niet belangrijk genoeg zijn, om te voorkomen dat je moet werken met zware datasets en grote grafieken. Bij het gebruik van Blackfire als debugging-tool is het echter beter om alle calls te behouden. Deze functionaliteit wordt door de debug addon toegevoegd.

Gebruik vanaf de command line de ``--debug`` parameter:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

In de productieomgeving zie je bijvoorbeeld dat een bestand genaamd ``.env.local.php`` wordt ingeladen:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

Waar komt dit vandaan? SymfonyCloud doet enkele optimalisaties bij het deployen van een Symfony applicatie, zoals het optimaliseren van de Composer autoloader ( ``--optimize-autoloader --apcu-autoloader --classmap-authoritative`` ). Ook de omgevingsvariabelen die in het ``.env`` bestand zijn gedefinieerd, worden geoptimaliseerd door het ``.env.local.php`` bestand te genereren (om te voorkomen dat het bestand bij elk verzoek opnieuw wordt geparsed):

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire is een zeer krachtige tool die je helpt inzicht te krijgen in hoe code wordt uitgevoerd door PHP. Het verbeteren van prestaties is slechts één manier om een profiler te gebruiken.
