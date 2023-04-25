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

Aby wygodnie i szybko wygenerować kontrolery, możemy skorzystać z pakietu ``symfony/maker-bundle``, który został zainstalowany, jako część paczki ``webapp``.

Maker Bundle pomaga wygenerować wiele różnych klas. Będziemy go używać w tej książce cały czas. Każdy "generator" jest definiowany w poleceniu i wszystkie polecenia należą do przestrzeni nazw ``make``.

.. index::
    single: Command;list

Wbudowane w Symfony Console polecenie ``list`` zawiera listę wszystkich poleceń dostępnych w danej przestrzeni nazw; użyj go, aby odkryć wszystkie generatory dostarczone przez Maker Bundle:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

Wybieranie formatu konfiguracji
-------------------------------

Przed stworzeniem pierwszego kontrolera w projekcie, musimy wybrać format konfiguracyjny, który chcemy wykorzystać. Symfony obsługuje YAML, XML, PHP i atrybuty PHP od razu po instalacji.

Do *konfiguracji pakietów* najlepszym wyborem jest *YAML*. Pliki konfiguracyjne znajdują się w katalogu ``config/``. Często, gdy instalujesz nowy pakiet, przepis pakietu (ang. recipe) podczas procesu instalacyjnego doda w tym katalogu nowy plik kończący się rozszerzeniem ``.yaml``.

W przypadku *konfiguracji kodu PHP* *atrybuty* są lepszym wyborem, ponieważ są definiowane w ramach kodu. Pozwól, że wyjaśnię to na przykładzie. Kiedy przychodzi żądanie, konfiguracja musi powiedzieć Symfony, że ścieżka żądania powinna być obsługiwana przez określony kontroler (klasa PHP). W przypadku korzystania z formatów konfiguracyjnych YAML, XML lub PHP, potrzebne są dwa pliki (plik konfiguracyjny i plik kontrolera PHP). W przypadku korzystania z atrybutów konfiguracja odbywa się bezpośrednio w klasie kontrolera.

Możesz się zastanawiać, jak odgadnąć nazwę pakietu wymaganego przez dane rozszerzenie? Nie musisz jej znać, gdyż - w większości przypadków - Symfony zawiera nazwę niezbędnego pakietu w komunikatach o błędach. Na przykład uruchamianie ``symfony console make:message`` bez pakietu ``messenger`` zakończyłoby się wyjątkiem zawierającym podpowiedź o potrzebie zainstalowania właściwego pakietu.

Generowanie kontrolera
----------------------

.. index::
    single: Command;make:controller

Utwórz swój pierwszy *kontroler* za pomocą polecenia ``make:controller``:

.. code-block:: terminal

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

Polecenie tworzy klasę ``ConferenceController`` w katalogu ``src/Controller/``. Wygenerowana klasa zawiera gotowy kod, który możesz modyfikować:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

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

Atrybut ``#[Route('/conference', name: 'app_conference')]`` jest tym, co czyni metodę ``index()`` kontrolerem (konfiguracja znajduje się nad kodem, który konfiguruje).

Po wejściu na adres: ``/conference`` w przeglądarce kontroler jest wykonywany, a odpowiedź zwracana.

Dostosuj trasę (ang. route), aby kierowała do strony głównej:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

Parametr nazwy trasy (ang. route) - ``name`` - będzie przydatny, gdy chcemy odnieść się do strony głównej w kodzie. Zamiast kodowania ścieżki ``/`` na sztywno, użyjemy nazwy trasy.

Zamiast domyślnie renderowanej strony, zwróćmy prostą stronę HTML:

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

Odśwież przeglądarkę:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

Głównym zadaniem kontrolera jest zwrócenie ``odpowiedzi`` HTTP dla żądania.

Ponieważ reszta rozdziału dotyczy kodu, którego nie zachowamy, zatwierdźmy teraz nasze zmiany:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

Dodawanie Easter Egga
---------------------

Aby zademonstrować, w jaki sposób odpowiedź może wykorzystać informacje z żądania, dodajmy małego `easter egga`_. Ilekroć strona główna zawiera łańcuch zapytań (ang. query string) taki jak: ``?hello=Fabien``, dodajmy tekst, aby powitać osobę:

.. code-block:: diff
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

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

Symfony udostępnia dane żądania poprzez obiekt ``Request``. Kiedy widzi argument kontrolera z typem ``Request``, automatycznie wie, jaki obiekt przekazać. Może zostać wykorzystany, aby pobrać element ``name`` z łańcucha zapytań i dodać tytuł ``<h1>``.

Otwórz w przeglądarce adres ``/`` a następnie ``/?hello=Fabien`` i zobacz różnice

.. note::

    Zwróć uwagę na wywołanie funkcji ``htmlspecialchars()``, dzięki czemu unikniemy problemów związanych z XSS. Jest to operacja, która zostanie dla nas wykonana automatycznie po przełączeniu się na odpowiedni silnik szablonów.

Mogliśmy również uczynić z nazwy część adresu URL:

.. code-block:: diff

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Annotation\Route;

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

Część ``{name}`` jest dynamicznym *parametrem trasy* (ang. route parameter) - działa jak symbol wieloznaczny (ang. wildcard). Możesz teraz odwiedzić ``/hello/Fabien`` w przeglądarce, aby uzyskać takie same wyniki jak poprzednio. *Wartość* ``{name}`` parametru można uzyskać poprzez dodanie argumentu kontrolera o tej samej nazwie  - ``$name``.

Cofnij zmiany, które właśnie wprowadziliśmy:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

Debugowanie zmiennych
---------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Świetnym narzędziem debugowania jest funkcja Symfony ``dump()``. Jest zawsze dostępna i pozwala zrzucać zmienne w ładnym i interaktywnym formacie.

Zmień na chwilę ``src/Controller/ConferenceController.php``, aby zrzucić obiekt Request:

.. code-block:: diff
    :emphasize-lines: 17

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

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

Podczas odświeżania strony, zwróć uwagę na nową ikonę „celu” na pasku narzędzi; pozwala ona sprawdzić zmienne, który debugujesz. Kliknij ją, aby uzyskać dostęp do pełnej strony, na której nawigacja jest prostsza:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Cofnij zmiany, które właśnie wprowadziliśmy:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: Idąc dalej

    * System `trasowania`_ Symfony;

    * `Samouczek SymfonyCasts: routing, kontrolery i strony`_;

    * `Atrybuty w PHP`_;

    * Komponent `HttpFoundation`_;

    * Ataki `XSS (Cross-Site Scripting)`_;

    * `Ściągawka Symfony Routing`_.

.. _`easter egga`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`trasowania`: https://symfony.com/doc/current/routing.html
.. _`Samouczek SymfonyCasts: routing, kontrolery i strony`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`Atrybuty w PHP`: https://www.php.net/attributes
.. _`HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (Cross-Site Scripting)`: https://owasp.org/www-community/attacks/xss/
.. _`Ściągawka Symfony Routing`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
