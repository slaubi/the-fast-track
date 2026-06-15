Een controller aanmaken
=======================

.. index::
    single: Controller
    single: Routing;Route

Ons gastenboek-project is al live op productieservers, maar we hebben een beetje vals gespeeld. Het project heeft nog geen webpagina's. De homepage is een saaie 404-foutpagina. Laten we dat oplossen.

Wanneer er een HTTP-request binnenkomt, zoals voor de homepage ( ``http://localhost:8000/`` ), probeert Symfony een *route* te vinden die overeenkomt met het *aanvraagpad* ( ``/`` hier). Een *route* is de link tussen het aanvraagpad en een *PHP callable* , een functie die de *HTTP-response* voor deze aanroep creëert.

Deze callables worden "controllers" genoemd. In Symfony zijn de meeste controllers geïmplementeerd als PHP-classes. Je kunt zo'n class handmatig maken, maar omdat we graag tempo maken, laten we eens kijken hoe Symfony ons kan helpen.

Lui zijn met de Maker Bundle
----------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Om moeiteloos controllers te genereren, kunnen we de ``symfony/maker-bundle``-package gebruiken, die geïnstalleerd werd als onderdeel van de ``webpack`` package.

De maker bundle helpt je om veel verschillende classes te genereren. We zullen dit dan ook constant gebruiken in dit boek. Elke "generator" wordt gedefinieerd in een commando en alle commando's maken deel uit van de ``make`` command namespace.

.. index::
    single: Command;list

Het ingebouwde ``list``-commando van de Symfony Console geeft een overzicht van alle commando's die beschikbaar zijn onder een bepaalde namespace; gebruik het om alle generatoren te ontdekken die door de maker bundle worden aangeleverd:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

Het kiezen van een configuratie-indeling
----------------------------------------

Voordat we de eerste controller van het project aanmaken, moeten we eerst beslissen welke configuratie-indeling we willen gebruiken. Symfony ondersteunt standaard YAML, XML, PHP en PHP attributen.

Voor *configuratie met betrekking tot packages* is *YAML* de beste keuze. Dit is het formaat dat in de ``config/``-directory wordt gebruikt. Wanneer je een nieuwe package installeert, zal het recipe van de package vaak een nieuw ``.yaml``-bestand toevoegen aan die map.

Voor *configuratie met betrekking tot PHP-code* zijn *attributen* een betere keuze omdat ze bij de code zijn gedefinieerd. Ik zal het uitleggen met een voorbeeld. Wanneer een request binnenkomt, moet een bepaalde configuratie Symfony vertellen dat het request path door een specifieke controller (een PHP-class) moet worden afgehandeld. Bij het gebruik van YAML-, XML- of PHP-configuratie-indeling gaat het om twee bestanden (het configuratiebestand en het PHP controller bestand). Bij het gebruik van attributen wordt de configuratie direct in de controller class gedaan.

Je vraagt je misschien af hoe je de packagenaam kunt raden die je voor een functionaliteit moet installeren? Meestal hoef je het niet te weten. In veel gevallen verwijst Symfony naar het te installeren package in zijn foutmeldingen. Wanneer je ``symfony make:message`` uitvoert zonder bijvoorbeeld het ``messenger``-package zal er een foutmelding optreden met daarin een hint over het installeren van het juiste package.

Een controller genereren
------------------------

.. index::
    single: Command;make:controller

Maak jouw eerste *controller* aan via het ``make:controller``-commando:

.. code-block:: terminal
    :class: answers(no)

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

Het commando creëert een ``ConferenceController``-class in de ``src/Controller/``-map. De gegenereerde class bestaat uit wat boilerplate code die klaar is om te worden uitgebreid:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;

    final class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'app_conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

De ``#[Route('/conference', name:'app_conference')]``-attribuut is wat de ``index()``-methode tot een controller maakt (de configuratie staat bij de code die dit configureert).

Wanneer je in een browser ``/conference`` bezoekt , wordt de controller uitgevoerd en wordt er een response teruggestuurd.

Pas de route aan zodat deze overeenkomt met de homepage:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

De route ``name`` is handig als we in de code naar de homepage willen verwijzen. In plaats van het ``/`` pad hard te coderen, gebruiken we de naam van de route.

Laten we een eenvoudige HTML-pagina terugsturen, in plaats van de standaard gerenderde pagina:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ final class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +            <html>
    +                <body>
    +                    <img src="/images/under-construction.gif" />
    +                </body>
    +            </html>
    +            EOF
    +        );
         }
     }

Vernieuw de browser:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

De hoofdverantwoordelijkheid van een controller is het terugsturen van een HTTP ``Response`` voor het verzoek.

Aan het einde van het hoofdstuk zullen we onze code wijzigingen terugdraaien. Laten we daarom onze wijzigingen nu committen:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

Een easter egg toevoegen
------------------------

Om aan te tonen hoe een response kan profiteren van de informatie uit het verzoek, laten we een kleine `Easter egg`_ toevoegen. Wanneer de homepage een query string bevat zoals ``?hello=Fabien``, voegen we wat tekst toe om de persoon te begroeten:

.. code-block:: diff
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
                 <html>
                     <body>
    +                    $greet
                         <img src="/images/under-construction.gif" />
                     </body>
                 </html>

Symfony maakt de gegevens van het verzoek beschikbaar via een ``Request`` object. Als Symfony een controller-argument ziet met deze class als typehint, weet het automatisch dat het dit argument aan jou door moet geven. We kunnen dit gebruiken om het ``name`` item uit de query string te halen en het aan een ``<h1>`` titel toe te voegen.

Bezoek dan ``/`` in een browser en vervolgens ``/?hello=Fabien`` om het verschil te zien.

.. note::

    Let op de call ``htmlspecialchars()`` om XSS-kwetsbaarheden te vermijden. Dit is iets dat automatisch voor ons zal worden gedaan wanneer we overschakelen naar een goede template engine.

We hadden ook de naam onderdeel kunnen maken van de URL:

.. code-block:: diff

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    -    public function index(Request $request): Response
    +    #[Route('/hello/{name}', name: 'homepage')]
    +    public function index(string $name = ''): Response
         {
             $greet = '';
    -        if ($name = $request->query->get('hello')) {
    +        if ($name) {
                 $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
             }

Het ``{name}`` deel van de route is een dynamische *routeparameter* - het werkt als een wildcard. Je kan nu ``/hello`` in een browser bezoeken en daarna ``/hello/Fabien`` om hetzelfde resultaat te krijgen. Je kan de *waarde* van de ``{name}`` parameter verkrijgen door een controllerparameter met dezelfde *naam* toe te voegen. Dus ``$name``.

Draai de wijzigingen die we net gemaakt hebben terug:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

Variabelen Debuggen
-------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Een geweldige debug helper is de Symfony ``dump()`` functie. Deze is altijd beschikbaar en laat het toe om ingewikkelde variabelen in een aangenaam en interactief formaat te dumpen.

Pas  tijdelijk ``src/Controller/ConferenceController.php`` aan om het Request object te dumpen:

.. code-block:: diff
    :emphasize-lines: 17

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    -    {
    +    public function index(Request $request): Response
    +        {
    +        dump($request);
    +
             return new Response(<<<EOF
                 <html>
                     <body>

Let op het nieuwe "target" icoontje in de toolbar wanneer we de pagina vernieuwen; let staat je toe om de dump te inspecteren. Klik er op om toegang te krijgen tot een volledige pagina waar je eenvoudiger kan navigeren:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Draai de wijzigingen die we net gemaakt hebben terug:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: Verder gaan

    * Het Symfony `routing`_-systeem;

    * `SymfonyCasts routes, controllers & pages tutorial`_;

    * `PHP attributen`_;

    * Het `HttpFoundation`_-component;

    * `XSS (Cross-Site Scripting)`_ beveiligingsaanvallen;

    * De `Symfony Routing Cheat Sheet`_.

.. _`Easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`routing`: https://symfony.com/doc/current/routing.html
.. _`SymfonyCasts routes, controllers & pages tutorial`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`PHP attributen`: https://www.php.net/attributes
.. _`HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (Cross-Site Scripting)`: https://owasp.org/www-community/attacks/xss/
.. _`Symfony Routing Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
