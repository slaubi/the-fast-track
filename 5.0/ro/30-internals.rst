Explorarea structurii interne Symfony
=====================================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

Folosim Symfony pentru a dezvolta o aplicație complexă de ceva vreme, dar cea mai mare parte a codului executat de aplicație provine de la Symfony. Câteva sute de linii de cod față de mii de linii de cod.

Îmi place să înțeleg cum funcționează lucrurile în culise. Și am fost întotdeauna fascinat de instrumente care mă ajută să înțeleg cum funcționează lucrurile. Prima dată când am folosit un depanator pas cu pas sau prima dată când am descoperit ``ptrace`` sunt amintiri magice.

Dorești să înțelegeți mai bine cum funcționează Symfony? E timpul să înțelegi modul în care Symfony face ca aplicația să funcționeze. În loc să descriem modul în care Symfony gestionează o solicitare HTTP dintr-o perspectivă teoretică, ceea ce ar fi destul de plictisitor, vom folosi Blackfire pentru a obține câteva reprezentări vizuale și pentru a descoperi anumite subiecte mai avansate.

Explorarea structurii interne Symfony cu Blackfire
--------------------------------------------------

Știi deja că toate cererile HTTP sunt difuzate de un singur punct de intrare: fișierul ``public/index.php``. Dar ce se întâmplă în continuare? Cum se numesc controlerele?

Să profilăm pagina de pornire în engleză în producție cu Blackfire prin extensia browserului Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

Sau direct prin linia de comandă:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Accesează vizualizarea „Timeline” a profilului, ar trebui să vezi ceva similar cu următoarele:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

Din cronologia de timp, navighează barele colorate pentru a avea mai multe informații despre fiecare apel; vei afla multe despre cum funcționează Symfony:

* Punctul principal de intrare este ``public/index.php``;

* Metoda ``Kernel::handle()`` gestionează cererea;

* Apelează ``HttpKernel`` care expediază anumite evenimente;

* Primul eveniment este ``RequestEvent``;

* Metoda ``ControllerResolver::getController()`` este apelată pentru a determina ce controler trebuie apelat pentru adresa URL primită;

* Metoda ``ControllerResolver::getArguments()`` este apelată să determine ce argumente să transmită controlerului (se numește convertorul de parametri);

* Se apelează metoda ``ConferenceController::index()`` și cea mai mare parte a codului nostru este executată de acest apel;

* Metoda ``ConferenceRepository::findAll()`` primește toate conferințele din baza de date (observă conexiunea la baza de date prin ``PDO::__construct()``);

* Metoda ``Twig\Environment::render()`` redă șablonul;

* ``ResponseEvent`` și ``FinishRequestEvent`` sunt expediate, dar se pare că niciun ascultător nu este de fapt înregistrat, deoarece se execută foarte rapid.

Cronologia este o modalitate excelentă de a înțelege modul în care funcționează un cod; ceea ce este foarte util când primești un proiect dezvoltat de altcineva.

Acum, profilează aceeași pagină din mașina locală din mediul de dezvoltare:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Deschide profilul. Ar trebui să fii redirecționat către vizualizarea graficului de apel, deoarece solicitarea a fost într-adevăr rapidă și cronologia ar fi destul de goală:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Înțelegi ce se întâmplă? Cache-ul HTTP este activat și, ca atare, profilăm stratul cache HTTP Symfony. Pe măsură ce pagina se află în cache, ``HttpCache\Store::restoreResponse()`` primește răspunsul HTTP din cache-ul său, iar controlerul nu este apelat niciodată.

Dezactivează stratul cache în ``public/index.php`` așa cum am făcut în pasul precedent și încearcă din nou. Poți vedea imediat că profilul arată foarte diferit:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Principalele diferențe sunt următoarele:

* ``TerminateEvent``, care nu era vizibil în producție, ia un procent mare din timpul de execuție; privind mai îndeaproape, poți vedea că acesta este evenimentul responsabil pentru stocarea datelor profilatorului Symfony adunate în timpul solicitării;

* Sub apelul ``ConferenceController::index()``, observă metoda ``SubRequestHandler::handle()`` care redă ESI (de aceea avem două apeluri la ``Profiler::saveProfile()``, una pentru cererea principală și alta pentru ESI).

Exploră cronologia pentru a afla mai multe; treci la vizualizarea graficului de apeluri pentru a avea o reprezentare diferită a acelorași date.

După cum tocmai am descoperit, codul executat în dezvoltare și producție este cu totul altul. Mediul de dezvoltare este mai lent, deoarece profilerul Symfony încearcă să adune multe date pentru a ușura problemele de depanare. Acesta este motivul pentru care ar trebui să profilezi întotdeauna cu mediul de producție, chiar și la nivel local.

Câteva experimente interesante: profilează o pagină de eroare, profilează pagina ``/`` (care este o redirecționare) sau o resursă API. Fiecare profil îți va spune un pic mai multe despre modul în care funcționează Symfony, ce clasă/metode sunt apelate, ce este scump de rulat și ce este ieftin.

Folosind addon-ul Blackfire Debug
---------------------------------

.. index::
    single: Blackfire;Debug Addon

În mod implicit, Blackfire elimină toate apelurile la metodă care nu sunt suficient de importante pentru a evita să aibă sarcini mari și grafice mari. Când utilizezi Blackfire ca instrument de depanare, este mai bine să păstrezi toate apelurile. Aceasta este furnizată de extensia de depanare.

Din linia de comandă, utilizează opțiunea ``--debug``:

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

În producție, vei observa, de exemplu, încărcarea unui fișier numit ``.env.local.php``:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

De unde vine? SymfonyCloud face unele optimizări atunci când implementează o aplicație Symfony, cum ar fi optimizarea autoloaderului Compozitorului (``--optimize-autoloader --apcu-autoloader --classmap-autoritative``). De asemenea, optimizează variabilele de mediu definite în fișierul ``.env`` (pentru a evita analizarea fișierului pentru fiecare solicitare) prin generarea fișierului ``.env.local.php``:

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire este un instrument foarte puternic care ajută la analizarea modului în care codul este executat de PHP. Îmbunătățirea performanței este doar o modalitate de a utiliza un profilator.
