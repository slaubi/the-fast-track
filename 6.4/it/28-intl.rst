Localizzazione di un'applicazione
=================================

Con un pubblico internazionale, Symfony è stato in grado di gestire l'internazionalizzazione (i18n) e la localizzazione (l10n) fin dalla prima versione. La localizzazione di un'applicazione non riguarda solo la traduzione dell'interfaccia, ma anche la gestione dei plurali, la formattazione delle date e delle valute, gli URL e altro ancora.

Internazionalizzazione degli URL
--------------------------------

.. index::
    single: Components;Routing
    single: Routing;Locale
    single: Routing;Requirements
    single: Attributes;Route

Il primo passo per internazionalizzare il sito è quello di internazionalizzare gli URL. Quando si traduce un'interfaccia di un sito, gli URL dovrebbero essere differenti a seconda della localizzazione per poter sfruttare la cache HTTP (non bisogna mai usare lo stesso URL per localizzazioni differenti e memorizzare la localizzazione nei dati di sessione).

Utilizzare il parametro speciale ``_locale`` per fare riferimento alla localizzazione nelle rotte:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
         ) {
         }

    -    #[Route('/', name: 'homepage')]
    +    #[Route('/{_locale}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Nella homepage, la localizzazione ora è impostata a seconda dell'URL; per esempio, se si visita la pagina ``/fr/``, il metodo ``$request->getLocale()`` restituisce il valore ``fr``.

Probabilmente non sarete in grado di tradurre i contenuti in tutte le lingue, limitatevi a quelle che volete supportare:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
         ) {
         }

    -    #[Route('/{_locale}/', name: 'homepage')]
    +    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Ogni parametro delle rotte può essere limitato da un'espressione regolare, inserita tra ``<`` e ``>``. La rotta ``homepage`` viene utilizzata solo quando il parametro ``_locale`` ha come valore ``en`` oppure ``fr``. Provate a visitare la pagina ``/es/`` e dovreste ricevere un errore 404 in quanto non esiste un rotta corrispondente.

Dato che useremo lo stesso requisito in quasi tutte le rotte, possiamo configurare il parametro nel container:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -9,6 +9,7 @@ parameters:
         admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"
         default_base_url: 'http://127.0.0.1'
         router.request_context.base_url: '%env(default:default_base_url:SYMFONY_DEFAULT_ROUTE_URL)%'
    +    app.supported_locales: 'en|fr'

     services:
         # default configuration for services in *this* file
    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ class ConferenceController extends AbstractController
         ) {
         }

    -    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
    +    #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

Si può aggiungere una lingua nel parametro ``app.supported_languages``.

Aggiungere lo stesso prefisso alle rotte degli altri URL:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -35,7 +35,7 @@ class ConferenceController extends AbstractController
             ])->setSharedMaxAge(3600);
         }

    -    #[Route('/conference_header', name: 'conference_header')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/header.html.twig', [
    @@ -43,7 +43,7 @@ class ConferenceController extends AbstractController
             ])->setSharedMaxAge(3600);
         }

    -    #[Route('/conference/{slug}', name: 'conference')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,
             Conference $conference,

Abbiamo quasi finito. Non abbiamo più una rotta che corrisponde a ``/``. Aggiungiamola di nuovo facendola reindirizzare a ``/en/``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -27,6 +27,12 @@ class ConferenceController extends AbstractController
         ) {
         }

    +    #[Route('/')]
    +    public function indexNoLocale(): Response
    +    {
    +        return $this->redirectToRoute('homepage', ['_locale' => 'en']);
    +    }
    +
         #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {

Ora che tutte le rotte principali utilizzano la localizzazione, possiamo notare che gli URL generati per le pagine prendono automaticamente in considerazione la localizzazione corrente.

Aggiungere un selettore di localizzazione
-----------------------------------------

.. index::
    single: Twig;path
    single: Twig;Locale

Per consentire agli utenti di passare dalla localizzazione ``en``  predefinita a un'altra, aggiungiamo un selettore di localizzazione nell'intestazione:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
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

Per passare a un'altra localizzazione, passiamo esplicitamente il parametro ``_locale`` alla funzione ``path()``.

.. index::
    single: Twig;app.request
    single: Twig;locale_name

Aggiornare il template per visualizzare il nome della localizzazione corrente, sostituendo il valore "English":

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        English
    +        {{ app.request.locale|locale_name(app.request.locale) }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

``app`` è una variabile globale di Twig che dà accesso alla richiesta web corrente. Per convertire la localizzazione in una stringa leggibile, usiamo il filtro ``locale_name`` di Twig.

.. index::
    single: Components;String

A seconda della localizzazione, il nome della localizzazione visualizzata non è sempre in maiuscolo. Per trasformare le frasi con l'iniziale maiuscola, abbiamo bisogno di un filtro che gestisca i caratteri Unicode. Possiamo utilizzare il componente String (fornito da Symfony) e dalla sua integrazione con Twig:

.. code-block:: terminal

    $ symfony composer req twig/string-extra

.. index::
    single: Twig;u.title

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        {{ app.request.locale|locale_name(app.request.locale) }}
    +        {{ app.request.locale|locale_name(app.request.locale)|u.title }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

Ora è possibile passare dal francese all'inglese tramite il selettore e l'intera interfaccia si adatta di conseguenza piuttosto bene:

.. figure:: screenshots/intl-switcher.png
    :alt: /fr/conference/amsterdam-2019
    :align: center
    :figclass: with-browser

Traduzione dell'interfaccia
---------------------------

.. index::
    single: Components;Translation
    single: Translation
    single: Twig;trans

Tradurre ogni singola frase su di un sito web di grandi dimensioni può essere noioso, ma fortunatamente abbiamo solo una manciata di messaggi sul nostro sito. Cominciamo con tutte le frasi della homepage:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -20,7 +20,7 @@
                 <nav class="navbar navbar-expand-xl navbar-light bg-light">
                     <div class="container mt-4 mb-3">
                         <a class="navbar-brand me-4 pr-2" href="{{ path('homepage') }}">
    -                        &#128217; Conference Guestbook
    +                        &#128217; {{ 'Conference Guestbook'|trans }}
                         </a>

                         <button class="navbar-toggler border-0" type="button" data-bs-toggle="collapse" data-bs-target="#header-menu" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Show/Hide navigation">
    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
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

Il filtro ``trans`` di Twig permette di ottenere la traduzione di un valore nella localizzazione corrente. Se non viene trovata una traduzione, viene usata la localizzazione predefinita (*default locale*), come configurato in ``config/packages/translation.yaml``:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 2

    framework:
        default_locale: en
        translator:
            default_path: '%kernel.project_dir%/translations'
            fallbacks:
                - en

Si noti che la "scheda" della barra degli strumenti di debug, relativa alla traduzione, è diventata rossa:

.. figure:: screenshots/intl-wdt.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

Ci dice che ci sono 3 messaggi non ancora tradotti.

Fare clic sulla "scheda" per elencare tutti i messaggi per i quali Symfony non ha trovato una traduzione:

.. figure:: screenshots/intl-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

Aggiungere le traduzioni
------------------------

Come abbiamo visto nella configurazione in ``config/packages/translation.yaml``, le traduzioni sono memorizzate in una cartella ``translations/``, creata automaticamente.

Invece di creare manualmente i file di traduzione, possiamo utilizzare il comando ``translation:extract``:

.. code-block:: terminal

    $ symfony console translation:extract fr --force --domain=messages

Questo comando genera un file di traduzione (col parametro ``--force``) per la lingua ``fr`` e il contesto ``messages``. Il contesto ``messages`` contiene tutti i messaggi dell'applicazione escludendo quelli provenienti da Symfony stesso come gli errori di validazione o di sicurezza.

Modificare il file ``translations/messages+intl-icu.fr.xlf`` e tradurre i messaggi in francese. Non parli francese? Lascia che ti aiuti:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/translations/messages+intl-icu.fr.xlf
    +++ b/translations/messages+intl-icu.fr.xlf
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

Si noti che non tradurremo tutti i template, ma sentitevi liberi di farlo:

.. figure:: screenshots/intl-translated.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

Tradurre i form
---------------

.. index::
    single: Translation;Form
    single: Form;Translation

Le etichette dei form sono visualizzate automaticamente da Symfony tramite il sistema di traduzione. Andando alla pagina di una conferenza e cliccando sulla scheda "Translation" della barra degli strumenti di debug, dovremmo vedere tutte le etichette pronte per la traduzione:

.. figure:: screenshots/intl-form-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

Localizzare le date
-------------------

.. index::
    single: Localization
    single: Twig;format_datetime
    single: Twig;format_time
    single: Twig;format_date
    single: Twig;format_currency
    single: Twig;format_number

Se si passa al francese e si va su una pagina delle conferenze che contiene dei commenti, si noterà che le date dei commenti sono localizzate in modo automatico. Questo avviene grazie al filtro ``format_datetime`` di Twig, che utilizza la localizzazione (``{{ comment.createdAt|format_datetime('medium', 'short') }}``).

La localizzazione funziona per date, orari (``format_time``), valute (``format_currency``) e numeri in generale (``format_number``) come percentuali, durate, ortografia, ecc.

Tradurre i plurali
------------------

.. index::
    single: Translation;Plurals
    single: Translation;Conditions

La gestione dei plurali nelle traduzioni è un caso specifico del problema più generale della scelta di una traduzione basata su una condizione.

Nella pagina di una conferenza viene visualizzato il numero di commenti: ``There are 2 comments``. Per un singolo commento, viene visualizzato ``There are 1 comments``, che è sbagliato. Modifichiamo il template per convertire la frase in un messaggio traducibile:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -44,7 +44,7 @@
                             </div>
                         </div>
                     {% endfor %}
    -                <div>There are {{ comments|length }} comments.</div>
    +                <div>{{ 'nb_of_comments'|trans({count: comments|length}) }}</div>
                     {% if previous >= 0 %}
                         <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
                     {% endif %}

Per questo messaggio abbiamo utilizzato un'altra strategia di traduzione. Invece di mantenere la versione inglese nel template, l'abbiamo sostituito con un identificatore univoco. Questa strategia funziona meglio per un testo complesso e lungo.

Aggiorniamo il file di traduzione aggiungendo il nuovo messaggio:

.. code-block:: diff
    :caption: patch_file

    --- a/translations/messages+intl-icu.fr.xlf
    +++ b/translations/messages+intl-icu.fr.xlf
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

Non abbiamo ancora finito, perché ora dobbiamo aggiungere la traduzione in inglese. Creiamo il file ``translations/messages+intl-icu.en.xlf``:

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

Aggiornamento dei test funzionali
---------------------------------

Non dimentichiamo di aggiornare i test funzionali per riflettere i cambiamenti di contenuti e URL:

.. code-block:: diff
    :caption: patch_file

    --- a/tests/Controller/ConferenceControllerTest.php
    +++ b/tests/Controller/ConferenceControllerTest.php
    @@ -11,7 +11,7 @@ class ConferenceControllerTest extends WebTestCase
         public function testIndex()
         {
             $client = static::createClient();
    -        $client->request('GET', '/');
    +        $client->request('GET', '/en/');

             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
    @@ -20,7 +20,7 @@ class ConferenceControllerTest extends WebTestCase
         public function testCommentSubmission()
         {
             $client = static::createClient();
    -        $client->request('GET', '/conference/amsterdam-2019');
    +        $client->request('GET', '/en/conference/amsterdam-2019');
             $client->submitForm('Submit', [
                 'comment[author]' => 'Fabien',
                 'comment[text]' => 'Some feedback from an automated functional test',
    @@ -41,7 +41,7 @@ class ConferenceControllerTest extends WebTestCase
         public function testConferencePage()
         {
             $client = static::createClient();
    -        $crawler = $client->request('GET', '/');
    +        $crawler = $client->request('GET', '/en/');

             $this->assertCount(2, $crawler->filter('h4'));

    @@ -50,6 +50,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertPageTitleContains('Amsterdam');
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    -        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +        $this->assertSelectorExists('div:contains("There is one comment")');
         }
     }

.. sidebar:: Andare oltre

    * `Tradurre i messaggi utilizzando il formattatore ICU`_;

    * `Usare i filtri di traduzione di Twig`_.

.. _`Tradurre i messaggi utilizzando il formattatore ICU`: https://symfony.com/doc/current/translation/message_format.html
.. _`Usare i filtri di traduzione di Twig`: https://symfony.com/doc/current/translation/templates.html#translation-filters
