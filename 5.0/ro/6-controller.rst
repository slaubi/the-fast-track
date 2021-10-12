Crearea unui Controler
======================

.. index::
    single: Controller
    single: Routing;Route

Proiectul nostru al unei cărți de oaspeți este deja live pe serverele de producție, dar am trișat un pic. Proiectul nu are încă pagini web. Pagina principală este servită ca o pagină de eroare HTTP 404. Hai să rezolvăm asta.

Când vine o solicitare HTTP, de exemplu pentru pagina principală (``http: //localhost:8000/``), Symfony încearcă să găsească o *rută* care să se potrivească cu calea *solicitată* (în cazul de față ``/``). O *rută* este legătura dintre calea solicitată și o *funcție PHP* care creează *răspunsul* HTTP (HTTP Response) pentru solicitarea respectivă.

Aceste funcții apelabile sunt numite "controlere". În Symfony, majoritatea controlerelor sunt implementate în clase PHP. Poți crea o astfel de clasă manual, dar pentru că ne place să dezvoltăm mai rapid, să vedem cum ne poate ajuta Symfony.

Fiind leneș cu MakerBundle
---------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Pentru a genera controlere fără efort, putem folosi pachetul bundle ``symfony/maker-bundle``:

.. code-block:: bash

    $ symfony composer req maker --dev

Deoarece pachetul bundle *MakerBundle* este util doar în timpul dezvoltării, nu uita să activezi opțiunea ``--dev`` la *Composer* pentru a evita activarea lui și pentru producție.

Bundle-ul Maker te ajută să generezi o mulțime de clase diferite. Îl vom folosi tot timpul în această carte. Fiecare „generator” este definit într-o comandă și toate comenzile fac parte din spațiul de nume ``make``.

.. index::
    single: Command;list

Comanda ``list`` încorporată în consola Symfony listează toate comenzile disponibile sub un anumit spațiu de nume; folosește-l pentru a descoperi toate generatoarele furnizate de pachetul Maker:

.. code-block:: bash
    :class: ignore

    $ symfony console list make

Alegerea unui format de configurare
-----------------------------------

Înainte de a crea primul controler al proiectului, trebuie să decidem asupra formatelor de configurare pe care dorim să le utilizăm. Symfony acceptă YAML, XML, PHP și adnotări.

Pentru *configurația pachetelor*, *YAML* este cea mai bună alegere. Acesta este formatul utilizat în directorul ``config/``. Adesea, atunci când instalezi un pachet nou, rețeta acelui pachet va adăuga un nou fișier cu extensia ``.yaml`` în acel director.

Pentru *configurația legată de codul PHP*, *adnotările* sunt o alegere mai bună, deoarece sunt definite direct în cod. Permite-mi să-ți explic cu un exemplu: când cineva accesează un URL, în Symfony este creată o cerere (request), iar noi trebuie să configurăm un anumit controler (o clasă PHP) care să gestioneze o anumită cale. Când utilizezi formate de configurare YAML, XML sau PHP, sunt implicate două fișiere (fișierul de configurare și fișierul controler PHP). Când utilizezi adnotări, configurația se face direct în clasa controlerului.

Pentru a gestiona adnotările, trebuie să adăugăm o librărie ca dependență nouă:

.. code-block:: bash

    $ symfony composer req annotations

S-ar putea să te întrebi cum poți ghici numele pachetului de care ai nevoie pentru o anumită caracteristică? De cele mai multe ori, nu trebuie să știi. În multe cazuri, Symfony menționează pachetul de instalat în mesajele sale de eroare. Rularea ``symfony make:controller`` fără pachetul ``Annotations``, de exemplu, s-ar fi încheiat cu o excepție care conține un indiciu despre instalarea pachetului potrivit.

Generarea unui controler
------------------------

.. index::
    single: Command;make:controller

Creează primul tău *Controler* prin comanda ``make:controller``:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;@Route

Comanda creează o clasă ``ConferenceController`` sub directorul ``src/Controller/``. Clasa generată este are codul de bază pregătit pentru modificările tale:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 10

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class ConferenceController extends AbstractController
    {
        /**
         * @Route("/conference", name="conference")
         */
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

Adnotarea ``@Route ("/conference", name="conference")`` este ceea ce face din metoda ``index()`` un controler (configurația prin adnotare este plasată fix deasupra codului pe care îl configurează).

Dacă accesezi ``/conference`` într-un browser, așa ajunge controlerul să fie apelat și un răspuns să fie returnat.

Ajustează ruta pentru ca acesta să se potrivească cu pagina principală (prima pagină):

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,7 +9,7 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/conference", name="conference")
    +     * @Route("/", name="homepage")
          */
         public function index(): Response
         {

Proprietatea ``name`` a rutei va fi utilă atunci când dorim să facem referință la pagina principală în cod. În loc să scriem ``/``, vom folosi numele rutei.

În loc de pagina prestabilită, hai să returnăm un simplu cod HTML:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -13,8 +13,13 @@ class ConferenceController extends AbstractController
          */
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +<html>
    +    <body>
    +        <img src="/images/under-construction.gif" />
    +    </body>
    +</html>
    +EOF
    +        );
         }
     }

Reîncarcă pagina în browser:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Responsabilitatea principală a unui controler este de a returna un HTTP ``Response`` pentru cererea primită.

.. _easter-egg:

Adăugarea unei surprize
------------------------

Pentru a demonstra modul în care un răspuns poate folosi informațiile din cererea primită, să adăugăm o mică `surpriză`_: să salutăm persoana care intră pe pagină ori de câte ori există un parametru GET de genul ``?hello=Fabien``:

.. code-block:: diff
    :emphasize-lines: 16
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,6 +3,7 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

    @@ -11,11 +12,17 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/", name="homepage")
          */
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
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Symfony expune datele cererii printr-un obiect de tip ``Request``. Când Symfony vede un argument al controlerului cu această definiție, știe să-l injecteze automat. Îl putem folosi pentru a obține elementul ``name`` din parametrul GET și pentru a adăuga un titlu ``<h1>`` în HTML.

Încearcă să încarci în browser ``/`` și apoi `` /?hello=Fabien`` pentru a vedea diferența.

.. note::

    Observă apelul la ``htmlspecialchars()`` pentru a evita problemele XSS. Este ceva ce se va face automat atunci când vom folosi un *motor de șabloane* adecvat.

De asemenea, am fi putut seta numele ca parte din URL:

.. code-block:: diff
    :emphasize-lines: 8,11
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,13 +9,19 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
         /**
    -     * @Route("/", name="homepage")
    +     * @Route("/hello/{name}", name="homepage")
          */
    -    public function index(): Response
    +    public function index(string $name = ''): Response
         {
    +        $greet = '';
    +        if ($name) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Parametrul ``{name}`` al căii este un parametru de rută *dinamic* - funcționează ca un *wildcard*. Acum poți accesa într-un browser ``/hello``, apoi ``/hello/Fabien`` pentru a obține aceleași rezultate ca înainte. Poți obține *valoarea* parametrului ``{name}`` adăugând un argument în controler cu același *name*. Deci ``$name``.

.. sidebar:: Mergând mai departe

    * Sistemul de rutare Symfony `Routing <https://symfony.com/doc/current/routing.html>`_;

    * `SymfonyCasts: tutorial pentru rute, controlere și pagini <https://symfonycasts.com/screencast/symfony/route-controller>`_;

    * `Adnotări <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_ în PHP;

    * Componenta `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_;

    * `Atacuri de securitate XSS (Cross-Site Scripting) <https://www.owasp.org/index.php/Cross-site_Scripting_ (XSS)>`_;

    * `Referințe Symfony Routing <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`surpriză`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
