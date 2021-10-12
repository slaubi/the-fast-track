Tworzenie kontrolera
====================

.. index::
    single: Controller
    single: Routing;Route

Nasz projekt księgi gości jest już dostępny na serwerach produkcyjnych, ale trochę oszukiwaliśmy. Projekt nie posiada jeszcze żadnych podstron. Za stronę główną służy nudny błąd 404. Naprawmy to.

Kiedy przychodzi żądanie HTTP, np. dla strony głównej (``http://localhost:8000/``), Symfony próbuje znaleźć *trasę* (ang. route), która pasuje do *ścieżki zapytania* (tutaj ``/``). *Trasa* jest łącznikiem pomiędzy ścieżką zapytania i *wywoływaczem PHP* (ang. PHP callable), funkcją, która tworzy *odpowiedź* dla tego żądania HTTP.

Te wywoływacze (ang. callable) nazywane są "kontrolerami". W Symfony większość kontrolerów jest zaimplementowana jako klasy PHP. Możesz stworzyć taką klasę ręcznie, ale ponieważ lubimy działać szybko, zobaczmy, jak Symfony może nam pomóc.

Lenistwo zawdzięczane Maker Bundle
-----------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Aby wygodnie i szybko wygenerować kontrolery, możemy skorzystać z pakietu ``symfony/maker-bundle``:

.. code-block:: bash

    $ symfony composer req maker --dev

Ponieważ pakiet ten jest użyteczny tylko podczas kodowania, nie zapomnij dodać flagi ``--dev``, aby uniknąć włączenia go w środowisku produkcyjnym.

Maker Bundle pomaga wygenerować wiele różnych klas. Będziemy go używać w tej książce cały czas. Każdy "generator" jest definiowany w poleceniu i wszystkie polecenia należą do przestrzeni nazw ``make``.

.. index::
    single: Command;list

Wbudowane w Symfony Console polecenie ``list`` zawiera listę wszystkich poleceń dostępnych w danej przestrzeni nazw; użyj go, aby odkryć wszystkie generatory dostarczone przez Maker Bundle:

.. code-block:: bash
    :class: ignore

    $ symfony console list make

Wybieranie formatu konfiguracji
-------------------------------

Przed stworzeniem pierwszego kontrolera w projekcie, musimy wybrać format konfiguracyjny, który chcemy wykorzystać. Symfony obsługuje YAML, XML, PHP i adnotacje od razu po instalacji.

Do *konfiguracji pakietów* najlepszym wyborem jest *YAML*. Pliki konfiguracyjne znajdują się w katalogu ``config/``. Często, gdy instalujesz nowy pakiet, przepis pakietu (ang. recipe) podczas procesu instalacyjnego doda w tym katalogu nowy plik kończący się rozszerzeniem ``.yaml``.

W przypadku *konfiguracji kodu PHP* *adnotacje* są lepszym wyborem, ponieważ są definiowane w ramach kodu. Pozwól, że wyjaśnię to na przykładzie. Kiedy przychodzi żądanie, konfiguracja musi powiedzieć Symfony, że ścieżka żądania powinna być obsługiwana przez określony kontroler (klasa PHP). W przypadku korzystania z formatów konfiguracyjnych YAML, XML lub PHP, potrzebne są dwa pliki (plik konfiguracyjny i plik kontrolera PHP). W przypadku korzystania z adnotacji konfiguracja odbywa się bezpośrednio w klasie kontrolera.

Aby wykorzystać adnotacje, musimy dodać kolejną zależność:

.. code-block:: bash

    $ symfony composer req annotations

Możesz się zastanawiać, jak odgadnąć nazwę pakietu wymaganego przez dane rozszerzenie? Nie musisz jej znać, gdyż - w większości przypadków - Symfony zawiera nazwę niezbędnego pakietu w komunikatach o błędach. Na przykład uruchamianie ``symfony make:controller`` bez pakietu ``annotations`` zakończyłoby się wyjątkiem zawierającym podpowiedź o potrzebie zainstalowania właściwego pakietu.

Generowanie kontrolera
----------------------

.. index::
    single: Command;make:controller

Utwórz swój pierwszy *kontroler* za pomocą polecenia ``make:controller``:

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;@Route

Polecenie tworzy klasę ``ConferenceController`` w katalogu ``src/Controller/``. Wygenerowana klasa zawiera gotowy kod, który możesz modyfikować:

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

Adnotacja ``@Route("/conference", name="conference")`` jest tym, co czyni metodę ``index()`` kontrolerem (konfiguracja znajduje się nad kodem, który konfiguruje).

Po wejściu na adres: ``/conference`` w przeglądarce kontroler jest wykonywany, a odpowiedź zwracana.

Dostosuj trasę (ang. route), aby kierowała do strony głównej:

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

Parametr nazwy trasy (ang. route) - ``name`` - będzie przydatny, gdy chcemy odnieść się do strony głównej w kodzie. Zamiast kodowania ścieżki ``/`` na sztywno, użyjemy nazwy trasy.

Zamiast domyślnie renderowanej strony, zwróćmy prostą stronę HTML:

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

Odśwież przeglądarkę:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Głównym zadaniem kontrolera jest zwrócenie ``odpowiedzi`` HTTP dla żądania.

.. _easter-egg:

Dodawanie Easter Egga
---------------------

Aby zademonstrować, w jaki sposób odpowiedź może wykorzystać informacje z żądania, dodajmy małego `easter egga`_. Ilekroć strona główna zawiera łańcuch zapytań (ang. query string) taki jak: ``?hello=Fabien``, dodajmy tekst, aby powitać osobę:

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

Symfony udostępnia dane żądania poprzez obiekt ``Request``. Kiedy widzi argument kontrolera z typem ``Request``, automatycznie wie, jaki obiekt przekazać. Może zostać wykorzystany, aby pobrać element ``name`` z łańcucha zapytań i dodać tytuł ``<h1>``.

Otwórz w przeglądarce adres ``/`` a następnie ``/?hello=Fabien`` i zobacz różnice

.. note::

    Zwróć uwagę na wywołanie funkcji ``htmlspecialchars()``, dzięki czemu unikniemy problemów związanych z XSS. Jest to operacja, która zostanie dla nas wykonana automatycznie po przełączeniu się na odpowiedni silnik szablonów.

Mogliśmy również uczynić z nazwy część adresu URL:

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

Część ``{name}`` jest dynamicznym *parametrem trasy* (ang. route parameter) - działa jak symbol wieloznaczny (ang. wildcard). Możesz teraz odwiedzić ``/hello/Fabien`` w przeglądarce, aby uzyskać takie same wyniki jak poprzednio. *Wartość* ``{name}`` parametru można uzyskać poprzez dodanie argumentu kontrolera o tej samej nazwie  - ``$name``.

.. sidebar:: Idąc dalej

    * System `trasowania <https://symfony.com/doc/current/routing.html>`_ Symfony;

    * `Samouczek SymfonyCasts: routing, kontrolery i strony <https://symfonycasts.com/screencast/symfony/route-controller>`_;

    * `Adnotacje <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_; w PHP;

    * Komponent `HttpFoundation; <https://symfony.com/doc/current/components/http_foundation.html>`_

    * Ataki `XSS (Cross-Site Scripting) <https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)>`_;

    * `Ściągawka Symfony Routing <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`easter egga`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
