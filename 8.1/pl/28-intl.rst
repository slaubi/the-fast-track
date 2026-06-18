Lokalizacja aplikacji
=====================

Dzięki użytkownikom z całego świata, Symfony od początku istnienia jest w stanie poradzić sobie z internacjonalizacją (i18n) i lokalizacją (l10n) od razu po zainstalowaniu frameworka. Lokalizacja aplikacji to nie tylko tłumaczenie interfejsu, ale także uwzględnienie liczby mnogiej tłumaczonych rzeczowników, formatowania dat i walut, adresów URL i wiele więcej.

Internacjonalizacja adresów URL
--------------------------------

.. index::
    single: Components;Routing
    single: Routing;Locale
    single: Routing;Requirements
    single: Attributes;Route

Pierwszym krokiem do internacjonalizacji strony internetowej jest internacjonalizacja adresów URL. Podczas tłumaczenia interfejsu strony internetowej, adres URL powinien być inny dla każdego miejsca, aby współgrał z pamięcią podręczną HTTP (nigdy nie przechowuj adresu URL razem z kodem języka w sesji).

Użyj specjalnego parametru trasy (ang. route), ``_locale``, aby odnieść się do lokalizacji w trasach:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/', name: 'homepage')]
    +    #[Route('/{_locale}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Kod języka (ang. locale) na stronie głównej jest teraz ustawiany wewnętrznie w zależności od adresu URL; na przykład, jeśli trafisz ``/fr/``, ``$request->getLocale()`` zwraca ``fr``.

Ponieważ prawdopodobnie nie będziesz w stanie przetłumaczyć treści na wszystkie dostępne języki, ogranicz się do tych, które chcesz obsługiwać:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/{_locale}/', name: 'homepage')]
    +    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Każdy parametr trasy może być ograniczony wyrażeniem regularnym wewnątrz ``<`` oraz ``>``. Trasa ``homepage`` pasuje tylko wtedy, gdy  parametr ``_locale`` jest ustawiony na ``en`` lub ``fr``. Podaj URL ``/es/``, a dostaniesz błąd 404, ponieważ żadna trasa nie została dopasowana do wyrażenia regularnego.

Ponieważ na prawie wszystkich trasach będziemy stosować ten sam wymóg, przenieśmy je do parametru kontenera:

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -9,5 +9,6 @@ parameters:
         admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"
         default_base_url: 'http://127.0.0.1'
    +    app.supported_locales: 'en|fr'

     services:
         # default configuration for services in *this* file
    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
    +    #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Język można dodać poprzez modyfikację parametru ``app.supported_languages``.

Dodaj ten sam prefiks trasy dla danego kraju do innych adresów URL:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -38,7 +38,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/conference_header', name: 'conference_header')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/header.html.twig', [
    @@ -46,8 +46,8 @@ final class ConferenceController extends AbstractController
             ]);
         }

         #[RateLimit('comment_submission', methods: ['POST'])]
    -    #[Route('/conference/{slug:conference}', name: 'conference')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,
             Conference $conference,

Prawie skończyliśmy. Nie mamy już trasy, która pasowałaby do ``/``. Dodajmy ją z powrotem i przekierujmy na ``/en/``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -27,6 +27,12 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Route('/')]
    +    public function indexNoLocale(): Response
    +    {
    +        return $this->redirectToRoute('homepage', ['_locale' => 'en']);
    +    }
    +
         #[Cache(smaxage: 3600)]
         #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response

Teraz, gdy wszystkie główne trasy biorą pod uwagę obecne ustawienia regionalne, należy zauważyć, że wygenerowane adresy URL na stronach automatycznie uwzględniają kod języka.

Dodawanie przełącznika ustawień regionalnych (ang. locale)
-------------------------------------------------------------

.. index::
    single: Twig;path
    single: Twig;Locale

Aby umożliwić użytkownikom przejście z domyślnego kodu języka (ang. locale) ``en`` na inny, dodajmy przełącznik w nagłówku:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -34,6 +34,16 @@
                                         Admin
                                     </a>
                                 </li>
    +<li class="nav-item dropdown">
    +    <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
    +        data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    +        English
    +    </a>
    +    <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
    +        <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>
    +        <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'fr'}) }}">Français</a></li>
    +    </ul>
    +</li>
                             </ul>
                         </div>
                     </div>

Aby przełączyć się na inny kod języka (ang. locale), przekazujemy parametr ``_locale`` trasy do funkcji ``path()``.

.. index::
    single: Twig;app.request
    single: Twig;locale_name

Zaktualizuj szablon, aby wyświetlić aktualną nazwę języka zamiast zakodowanego na sztywno "English":

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        English
    +        {{ app.request.locale|locale_name(app.request.locale) }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

``app`` jest globalną zmienną biblioteki Twig, która umożliwia dostęp do bieżącego żądania (ang. request). Aby przekonwertować nazwę obecnej lokalizacji do formy czytelnej dla użytkownika, używamy filtra ``locale_name``.

.. index::
    single: Components;String

W zależności od języka, nazwa nie zawsze jest zapisana wielkimi literami. Do poprawnej zmiany wielkości liter w zdaniu potrzebny jest filtr, który uwzględnia Unicode. Zapewnia go komponent Symfony String i jego implementacja w Twigu:

.. code-block:: terminal

    $ symfony composer req twig/string-extra

.. index::
    single: Twig;u.title

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        {{ app.request.locale|locale_name(app.request.locale) }}
    +        {{ app.request.locale|locale_name(app.request.locale)|u.title }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

Teraz można zmienić język z francuskiego na angielski za pomocą przełącznika, a cały interfejs ładnie się dostosowuje:

.. figure:: screenshots/intl-switcher.png
    :alt: /fr/conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Tłumaczenie interfejsu
-----------------------

.. index::
    single: Components;Translation
    single: Translation
    single: Twig;trans

Tłumaczenie każdego zdania na dużej stronie internetowej może być żmudne, ale na szczęście u nas jest tylko kilka wiadomości. Zacznijmy od wszystkich zdań na stronie głównej:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -20,7 +20,7 @@
                 <nav class="navbar navbar-expand-xl navbar-light bg-light">
                     <div class="container mt-4 mb-3">
                         <a class="navbar-brand me-4 pr-2" href="{{ path('homepage') }}">
    -                        &#128217; Conference Guestbook
    +                        &#128217; {{ 'Conference Guestbook'|trans }}
                         </a>

                         <button class="navbar-toggler border-0" type="button" data-bs-toggle="collapse" data-bs-target="#header-menu" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Show/Hide navigation">
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -4,7 +4,7 @@

     {% block body %}
         <h2 class="mb-5">
    -        Give your feedback!
    +        {{ 'Give your feedback!'|trans }}
         </h2>

         {% for row in conferences|batch(4) %}
    @@ -21,7 +21,7 @@

                                 <a href="{{ path('conference', { slug: conference.slug }) }}"
                                    class="btn btn-sm btn-primary stretched-link">
    -                                View
    +                                {{ 'View'|trans }}
                                 </a>
                             </div>
                         </div>

Filtr ``trans`` biblioteki Twig wyszukuje tłumaczenie podanego tekstu na bieżącą lokalizację. Jeśli nie zostanie znalezione, powróci do wartości z *domyślnej lokalizacji* skonfigurowanej w ``config/packages/translation.yaml``:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 2

    framework:
        default_locale: en
        translator:
            default_path: '%kernel.project_dir%/translations'
            fallbacks:
                - en

Zauważ, że na pasku narzędzi do debugowania zakładka tłumaczeń zmieniła kolor na czerwony:

.. figure:: screenshots/intl-wdt.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

To jest sygnał, że trzy wiadomości nie zostały jeszcze przetłumaczone.

Kliknij na zakładkę, aby wyświetlić listę wszystkich wiadomości, dla których Symfony nie znalazł tłumaczenia:

.. figure:: screenshots/intl-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

Dostarczanie tłumaczeń
------------------------

Jak można było zauważyć w ``config/packages/translation.yaml``, tłumaczenia są przechowywane w katalogu ``translations/``,  który został utworzony automatycznie.

Zamiast tworzyć pliki tłumaczeń ręcznie, użyj polecenia ``translation:extract``:

.. code-block:: terminal

    $ symfony console translation:extract fr --force --domain=messages

Polecenie to generuje plik tłumaczeń (flaga ``--force``) dla kodu języka ``fr`` i domeny ``messages``, która zawiera wszystkie wiadomości związane z **aplikacją**, ale bez tych, które dostarcza Symfony – takich jak związane z walidacją lub błędami bezpieczeństwa.

Edytuj plik ``translations/messages+intl-icu.fr.xlf`` i przetłumacz zawarte w nim teksty na język francuski. Nie mówisz po francusku? Pozwól, że Ci pomogę:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/translations/messages+intl-icu.fr.xlf
    +++ w/translations/messages+intl-icu.fr.xlf
    @@ -7,15 +7,15 @@
         <body>
           <trans-unit id="eOy4.6V" resname="Conference Guestbook">
             <source>Conference Guestbook</source>
    -        <target>__Conference Guestbook</target>
    +        <target>Livre d'Or pour Conferences</target>
           </trans-unit>
           <trans-unit id="LNAVleg" resname="Give your feedback!">
             <source>Give your feedback!</source>
    -        <target>__Give your feedback!</target>
    +        <target>Donnez votre avis !</target>
           </trans-unit>
           <trans-unit id="3Mg5pAF" resname="View">
             <source>View</source>
    -        <target>__View</target>
    +        <target>Sélectionner</target>
           </trans-unit>
         </body>
       </file>

.. code-block:: xml
    :caption: translations/messages+intl-icu.fr.xlf
    :class: hide

    <?xml version="1.0" encoding="utf-8"?>
    <xliff xmlns="urn:oasis:names:tc:xliff:document:1.2" version="1.2">
    <file source-language="en" target-language="fr" datatype="plaintext" original="file.ext">
        <header>
        <tool tool-id="symfony" tool-name="Symfony" />
        </header>
        <body>
        <trans-unit id="LNAVleg" resname="Give your feedback!">
            <source>Give your feedback!</source>
            <target>Donnez votre avis !</target>
        </trans-unit>
        <trans-unit id="3Mg5pAF" resname="View">
            <source>View</source>
            <target>Sélectionner</target>
        </trans-unit>
        <trans-unit id="eOy4.6V" resname="Conference Guestbook">
            <source>Conference Guestbook</source>
            <target>Livre d'Or pour Conferences</target>
        </trans-unit>
        </body>
    </file>
    </xliff>

Zauważ, że nie przetłumaczyłem wszystkich szablonów, ale możesz przejąć pałeczkę:

.. figure:: screenshots/intl-translated.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

Tłumaczenie formularzy
-----------------------

.. index::
    single: Translation;Form
    single: Form;Translation

Etykiety formularzy są automatycznie wyświetlane przez Symfony poprzez system tłumaczeń. Przejdź na stronę konferencji i kliknij na zakładkę "Tłumaczenie" na pasku narzędzi do debugowania; wszystkie etykiety powinny być gotowe do tłumaczenia:

.. figure:: screenshots/intl-form-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

Ustawienia regionalne dat
-------------------------

.. index::
    single: Localization
    single: Twig;format_datetime
    single: Twig;format_time
    single: Twig;format_date
    single: Twig;format_currency
    single: Twig;format_number

Jeżeli przełączysz się na francuski i wejdziesz na stronę konferencji, na której są już jakieś komentarze, zauważysz, że daty tych komentarzy są automatycznie dostosowane do ustawień regionalnych użytkownika. Ten mechanizm działa dzięki dostępnemu w Twigu filtrowi ``format_datetime``, który bierze pod uwagę ustawienia regionalne (``{{ comment.createdAt|format_datetime('medium', 'short') }}``).

Lokalizacja działa dla dat, czasu (``format_time``), walut (``format_currency``) i liczb (``format_number``) w wielu formach (procenty, czas trwania, zapis słowny i wiele innych).

Tłumaczenie liczby mnogiej
---------------------------

.. index::
    single: Translation;Plurals
    single: Translation;Conditions

Zarządzanie liczbą mnogą w tłumaczeniach jest jednym z przypadków bardziej ogólnego problemu wyboru tłumaczenia w oparciu o warunki.

Na stronie konferencji wyświetlana jest liczba komentarzy: ``There are 2 comments``. Dla jednego komentarza wyświetlamy ``There are 1 comments``, co jest błędne. Zmodyfikuj szablon, aby przekonwertować zdanie na komunikat do tłumaczenia:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -44,7 +44,7 @@
                             </div>
                         </div>
                     {% endfor %}
    -                <div>There are {{ comments|length }} comments.</div>
    +                <div>{{ 'nb_of_comments'|trans({count: comments|length}) }}</div>
                     {% if previous >= 0 %}
                         <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
                     {% endif %}

W tym celu zastosowaliśmy inną strategię tłumaczenia. Zamiast zachować wersję angielską w szablonie, zastąpiliśmy ją unikalnym identyfikatorem. Ta strategia lepiej sprawdza się w przypadku tekstów skomplikowanych i długich.

Zaktualizuj plik tłumaczenia poprzez dodanie nowej wiadomości:

.. code-block:: diff
    :caption: patch_file

    --- i/translations/messages+intl-icu.fr.xlf
    +++ w/translations/messages+intl-icu.fr.xlf
    @@ -17,6 +17,10 @@
             <source>Conference Guestbook</source>
             <target>Livre d'Or pour Conferences</target>
         </trans-unit>
    +    <trans-unit id="Dg2dPd6" resname="nb_of_comments">
    +        <source>nb_of_comments</source>
    +        <target>{count, plural, =0 {Aucun commentaire.} =1 {1 commentaire.} other {# commentaires.}}</target>
    +    </trans-unit>
         </body>
     </file>
     </xliff>

Jeszcze nie skończyliśmy, ponieważ musimy teraz zadbać o tłumaczenie na język angielski. Utwórz plik``translations/messages+intl-icu.en.xlf``:

.. code-block:: xml
    :caption: translations/messages+intl-icu.en.xlf
    :emphasize-lines: 10

    <?xml version="1.0" encoding="utf-8"?>
    <xliff xmlns="urn:oasis:names:tc:xliff:document:1.2" version="1.2">
      <file source-language="en" target-language="en" datatype="plaintext" original="file.ext">
        <header>
          <tool tool-id="symfony" tool-name="Symfony" />
        </header>
        <body>
          <trans-unit id="maMQz7W" resname="nb_of_comments">
            <source>nb_of_comments</source>
            <target>{count, plural, =0 {There are no comments.} one {There is one comment.} other {There are # comments.}}</target>
          </trans-unit>
        </body>
      </file>
    </xliff>

Aktualizowanie testów funkcjonalnych
-------------------------------------

Nie zapomnij zaktualizować testów funkcjonalnych, aby brały pod uwagę zmiany adresów URL i treści:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -16,7 +16,7 @@ class ConferenceControllerTest extends WebTestCase
         public function testIndex(): void
         {
             $client = static::createClient();
    -        $client->request('GET', '/');
    +        $client->request('GET', '/en/');

             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
    @@ -29,7 +29,7 @@ class ConferenceControllerTest extends WebTestCase
             $berlin = ConferenceFactory::createOne(['city' => 'Berlin', 'year' => '2021', 'isInternational' => false]);
             CommentFactory::createOne(['conference' => $berlin]);

    -        $client->request('GET', '/conference/berlin-2021');
    +        $client->request('GET', '/en/conference/berlin-2021');
             $client->submitForm('Submit', [
                 'comment[author]' => 'Fabien',
                 'comment[text]' => 'Some feedback from an automated functional test',
    @@ -50,7 +50,7 @@ class ConferenceControllerTest extends WebTestCase
             ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
             CommentFactory::createOne(['conference' => $amsterdam]);

    -        $crawler = $client->request('GET', '/');
    +        $crawler = $client->request('GET', '/en/');

             $this->assertCount(2, $crawler->filter('h4'));

    @@ -59,6 +59,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertPageTitleContains('Amsterdam');
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    -        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +        $this->assertSelectorExists('div:contains("There is one comment")');
         }
     }

.. sidebar:: Idąc dalej

    * `Tłumaczenie komunikatów z użyciem formatu ICU`_;

    * `Używanie filtrów tłumaczeniowych biblioteki Twig`_.

.. _`Tłumaczenie komunikatów z użyciem formatu ICU`: https://symfony.com/doc/current/translation/message_format.html
.. _`Używanie filtrów tłumaczeniowych biblioteki Twig`: https://symfony.com/doc/current/translation/templates.html#translation-filters
