Einen Controller erstellen
==========================

.. index::
    single: Controller
    single: Routing;Route

Unser Gästebuchprojekt ist bereits auf dem Server für den Produktivbetrieb live, aber wir haben ein wenig geschummelt: Das Projekt hat noch keine Webseiten. Die Homepage zeigt eine langweilige 404-Fehlerseite. Das wollen wir gleich korrigieren.

Wenn ein HTTP-Request (HTTP-Anfrage) eintrifft, wie z. B. für die Homepage (``http://localhost:8000/``), versucht Symfony, eine *Route* zu finden, der dem *Request-Pfad* entspricht (hier ``/``). Eine *Route* ist die Verbindung zwischen dem Request-Path (Anforderungspfad) und einem *PHP callable*  – einer Funktion, die die *HTTP-Response* (HTTP-Antwort) für diesen Request erzeugt.

Diese Callables werden als "Controller" bezeichnet. In Symfony sind die meisten Controller als PHP-Klassen implementiert. Du kannst eine solche Klasse manuell erstellen, aber da wir gerne schnell sind, wollen wir sehen, wie Symfony uns helfen kann.

Faul sein – mit dem Maker Bundle
----------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Um Controller mühelos zu generieren, können wir das ``symfony/maker-bundle``-Paket nutzen, welches mit dem ``webapp``-Paket installiert wurde.

Das Maker-Bundle hilft Dir, viele verschiedene Klassen zu generieren. Wir werden es in diesem Buch immer wieder verwenden. Jeder "Generator" ist in einem Befehl definiert und alle Befehle sind Teil des ``make``-Befehl-Namespaces.

.. index::
    single: Command;list

Der in der Symfony-Console eingebaute ``list``-Befehl listet alle Befehle auf, die unter einem bestimmten Namespace verfügbar sind; verwende ihn, um alle vom Maker-Bundle bereitgestellten Generatoren zu ermitteln:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

Ein Konfigurationsformat auswählen
-----------------------------------

Bevor wir den ersten Controller des Projekts erstellen, müssen wir uns entscheiden, welches Konfigurationsformat wir verwenden wollen. Symfony unterstützt YAML, XML, PHP und PHP-Attribute nativ.

Für die *Konfiguration in Bezug auf Pakete* ist *YAML* die beste Wahl. Dies ist das Format, das im ``config/`` Verzeichnis verwendet wird. Häufig, wenn Du ein neues Paket installierst, wird das Recipe dieses Pakets eine neue Datei mit der Endung in ``.yaml`` zu diesem Verzeichnis hinzufügen.

Für die *Konfiguration von PHP-Code* sind *Attribute* eine bessere Wahl, da sie nahe am Code hinterlegt sind. Lass mich das an einem Beispiel erklären: Wenn eine Request eingeht, muss eine Konfiguration Symfony mitteilen, dass der Requestpfad von einem bestimmten Controller (einer PHP-Klasse) behandelt werden soll. Bei der Verwendung von YAML-, XML- oder PHP-Konfigurationsformaten sind zwei Dateien beteiligt (die Konfigurationsdatei und die PHP-Controller-Datei). Bei der Verwendung von Attributen erfolgt die Konfiguration direkt in der Controller-Klasse.

Du fragst dich vielleicht, wie Du den Paketnamen erraten kannst, den Du für die Installation eines bestimmten Features eingeben musst? Meistens musst du es nicht wissen. In vielen Fällen enthält Symfony in seinen Fehlermeldungen das zu installierende Paket. Der Aufruf ``symfony make:message`` ohne das ``Messenger``-Paket zum Beispiel wäre mit einer Exception beendet worden, die einen Hinweis auf die Installation des richtigen Pakets enthält.

Einen Controller erzeugen
-------------------------

.. index::
    single: Command;make:controller

Erstelle Deinen ersten *Controller* über den ``make:controller`` Befehl:

.. code-block:: terminal

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

Der Befehl erstellt eine ``ConferenceController``-Klasse im ``src/Controller/``-Verzeichnis. Die generierte Klasse besteht aus Standard Code, der jetzt angepasst werden kann:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;

    class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'app_conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

Das ``#[Route('/conference', name: 'app_conference')]``-Attribut macht die ``index()``-Methode zu einem Controller (die Konfiguration befindet sich neben dem Code, den sie konfiguriert).

Wenn Du in einem Browser ``/conference`` aufrufst, wird der Controller ausgeführt und eine Response (Antwort) zurückgegeben.

Passe die Route an, damit sie mit der Homepage übereinstimmt:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Attribute\Route;

     class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

Der ``name`` der Route wird nützlich sein, wenn wir die Homepage im Code referenzieren wollen. Anstatt ``/`` als Pfad anzugeben (hard-coding), verwenden wir den Namen der Route.

Anstelle der standardmäßig gerenderten Seite geben wir eine einfache HTML-Seite zurück:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ class ConferenceController extends AbstractController
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

Aktualisiere den Browser:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Die Hauptaufgabe eines Controllers besteht darin, eine HTTP-``Response`` für den Request zurückzugeben.

Da der letzte Teil dieses Kapitels über Code ist, den wir nicht behalten werden, lass uns unser Änderungen committen.

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

Ein Easter egg hinzufügen
--------------------------

Um zu zeigen, wie eine Response Informationen aus dem Request nutzen kann, fügen wir ein kleines `Easter egg`_ hinzu. Wann immer die Homepage einen Query String wie ``?hello=Fabien`` enthält, fügen wir einen Text hinzu, um die Person zu begrüßen:

.. code-block:: diff
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     class ConferenceController extends AbstractController
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

Symfony stellt die Requestdaten über ein ``Request``-Objekt zur Verfügung. Wenn Symfony ein Controller-Argument mit diesem Type-Hint sieht, weiß es automatisch, dass es das ``Request``-Objekt  übergeben soll. Wir können es verwenden, um das ``name``-Element aus dem Query-String zu holen und zu einem ``<h1>``-Titel hinzu zufügen.

Versuche in einem Browser erst ``/`` und dann ``/?hello=Fabien`` aufzurufen, um den Unterschied zu sehen.

.. note::

    Beachte den Aufruf zu ``htmlspecialchars()``, um XSS-Probleme zu vermeiden. Sobald wir zu einer richtigen Template-Engine wechseln, wird das automatisch für uns erledigt.

Wir hätten den Namen auch in die URL aufnehmen können:

.. code-block:: diff

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Attribute\Route;

     class ConferenceController extends AbstractController
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

Der ``{name}``-Teil der Route ist ein dynamischer *Routenparameter* – er funktioniert wie ein Platzhalter. Du kannst nun ``/hello`` und dann ``/hello/Fabien`` in einem Browser eingeben, um die gleichen Ergebnisse wie zuvor zu erhalten. Du kannst den *Wert* des ``{name}``-Parameters erhalten, indem Du einen Controller Argument mit dem gleichen *Namen* hinzufügst. Also, ``$name``.

Mach die Änderungen, die wir gemacht haben, rückgängig:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

Debug Variablen
---------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Die Symfony``dump()``-Funktion ist ein großartiger Debug-Helfer. Er ist immer verfügbar und erlaubt Dir komplexe Variablen in einem guten und interaktiven Format zu exportieren.

Ändere ``src/Controller/ConferenceController.php`` für einen Moment um das Request-Objekt zu exportieren:

.. code-block:: diff
    :emphasize-lines: 17

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        dump($request);
    +
             return new Response(<<<EOF
                 <html>
                     <body>

Wenn Du die Seite aktualisierst, siehst Du ein neues "Zielscheiben"-Icon in der Toolbar; damit kannst Du den Export inspektieren. Wenn Du darauf klickst, hast Du Zugang zu einer ganzen Seite, wo es einfach ist durch zu navigatieren:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Mach die Änderungen, die wir gemacht haben, rückgängig:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: Weiterführendes

    * Das Symfony `Routing-System`_;

    * `SymfonyCasts-Tutorial für Routes, Controller & Seiten`_;

    * `PHP Attribute`_;

    * Die `HttpFoundation`_ Komponente;

    * `XSS (Cross-Site Scripting)`_ Sicherheitsangriffe;

    * Das `Symfony Routing Cheat Sheet`_.

.. _`Easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`Routing-System`: https://symfony.com/doc/current/routing.html
.. _`SymfonyCasts-Tutorial für Routes, Controller & Seiten`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`PHP Attribute`: https://www.php.net/attributes
.. _`HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (Cross-Site Scripting)`: https://owasp.org/www-community/attacks/xss/
.. _`Symfony Routing Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
