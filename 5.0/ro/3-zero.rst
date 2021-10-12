De la zero la producție
========================

Îmi place să mă mișc rapid. Vreau ca micul nostru proiect să fie în producție cât mai repede posibil. Acum, dacă se poate. Cum încă nu am dezvoltat nimic, vom începe prin implementarea unei simple și frumoase pagini „În construcție”. O să-ți placă!

Petrece ceva timp încercând să găsești GIF-ul animat „În construcție” ideal pe Internet. Iată `ce <http://clipartmag.com/images/website-under-construction-image-6.gif>`_ voi folosi:

.. image:: images/under-construction.gif
    :align: center

Ți-am zis, va fi distractiv.

Începutul proiectului
----------------------

Creează un nou proiect Symfony folosind utilitarul din linia de comandă ``symfony`` pe care l-am instalat anterior împreună:

.. code-block:: bash

    $ symfony new guestbook --version=5.0
    $ cd guestbook

Această comandă  împachetează ``Composer`` și ușurează crearea proiectelor Symfony. Folosește un `schelet de proiect <https://github.com/symfony/skeleton>`_ care include dependențele minime, componentele Symfony care sunt necesare pentru aproape orice proiect: un instrument pentru linia de comandă (Console) și abstractizarea HTTP necesară pentru a crea aplicații Web.

Dacă arunci o privire pe GitHub la schelet, vei observa că este aproape gol. Doar un fișier ``composer.json``. Dar directorul ``guestbook`` este plin de fișiere. Cum este posibil? Răspunsul se află în pachetul ``symfony/flex``. Symfony Flex este un modul pentru Composer care se conectează la procesul de instalare. Când detectează un pachet pentru care are o *rețetă*, îl execută.

Punctul principal al unei rețete Symfony este un fișier manifest care descrie operațiunile care trebuie făcute pentru a integra automat pachetul sau librăria într-o aplicație Symfony. Nu mai e nevoie să citești fișierul README pentru a instala un pachet în Symfony. Automatizarea este o caracteristică cheie a Symfony.

Dat fiind că avem Git instalat pe calculatorul nostru, ``symfony new`` a creat și un repozitoriu Git pentru noi, salvând prima modificare (primul commit).

Aruncă o privire la structura directoarelor:

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

Directorul ``bin/`` conține fișierul principal pentru linia de comandă: ``console``. Îl vei folosi tot timpul.

Directorul ``config/`` conține un un set de fișiere de configurare implicite și importante. Câte un fișier per librărie. N-o să fie nevoie să le schimbi prea des, de obicei valorile implicite sunt suficiente.

Directorul ``public/`` este directorul expus public, pe web, iar scriptul ``index.php`` este punctul principal de intrare pentru toate cererile HTTP.

În directorul ``src/`` este tot codul pe care îl vei scrie tu; aici vei petrece cea mai mare parte a timpului tău. În mod implicit, toate clasele din acest director folosesc calea (namespace) ``App``. Aici este casa pentru codul tău. Logica ta de *business*. Symfony nu îți impune mare lucru acolo.

Directorul ``var/`` conține fișierele temporare, cache, jurnalele (logs) și fișierele generate în timpul rulării de către aplicație. Poți să uiți de el. Este singurul director în care ai nevoie de permisiuni de scriere în producție.

Directorul ``vendor/`` conține toate pachetele instalate de Composer, inclusiv Symfony. Aici este arma noastră secretă pentru a fi mai productivi: să nu reinventăm roata. Te vei baza pe librăriile existente pentru a implementa funcționalități dificile. Directorul acesta este gestionat de Composer și nu ai de ce să faci modificări, manual, aici.

Asta este tot ce trebuie să știi deocamdată.

Crearea unor resurse publice
----------------------------

Orice este în folderul ``public/`` este accesibil printr-un browser. De exemplu, dacă muți fișierul GIF animat (denumește-l ``under-construction.gif``) într-un nou director ``public/images/``, acesta va fi disponibil pe o adresă URL precum ``https://localhost/images/under-construction.gif``.

Descarcă imaginea GIF de aici:

.. code-block:: bash

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Pornirea unui server web local
------------------------------

.. index::
    single: Symfony CLI;server:start

Utilitarul ``symfony`` din linia de comandă pune la dispoziție un server web optimizat pentru activități de dezvoltare. N-ar trebui să te surprindă faptul că funcționează bine cu Symfony. Totuși, nu-l folosi niciodată în producție.

Din directorul de proiect, pornește serverul web în fundal (cu parametrul ``-d``):

.. code-block:: bash

    $ symfony server:start -d

Serverul a pornit folosind primul port disponibil, portul 8000. Ca o scurtătură, poți deschide site-ul web într-un browser din linia de comandă:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Browserul tău preferat ar trebui să apară pe ecran și să deschidă o filă nouă care să afișeze ceva similar cu:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    Pentru a rezolva problemele, execută în linia de comandă ``symfony server:log``; acesta comandă afișează jurnalele (logs) de pe serverul web, din PHP și din aplicația ta.

Deschide în browser ``/images/under-construction.gif``. Arată așa?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

E bine? Atunci hai să salvăm ce-am făcut până acum:

.. code-block:: bash
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Adăugarea unui favicon
-----------------------

Pentru a evita multe erori HTTP 404 în jurnale, din cauza lipsei unui favicon solicitat de browsere, hai să adăugăm unul acum:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/favicon.ico', 'public/favicon.ico');"
    $ git add public/
    $ git commit -m'Add a favicon'

Pregătirea pentru producție
-----------------------------

.. index::
    single: SymfonyCloud;Initialization

Ce zici de punerea în funcțiune a aplicației noastre în producție? Știu că nu avem încă o pagină HTML adecvată pentru a le spune bun venit utilizatorilor noștri. Dar a vedea mica imagine „în construcție” pe un server de producție ar fi un mare pas înainte. Urmăm deviza: *lansează devreme și des*.

Poți găzdui această aplicație la orice furnizor de găzduire web care acceptă PHP... ceea ce înseamnă aproape toți furnizorii de găzduire existenți. Verifică totuși câteva lucruri: dorim cea mai recentă versiune PHP și posibilitatea de a găzdui servicii precum o bază de date, o coadă de mesaje și altele.

Eu am ales, și alegerea mea va fi `SymfonyCloud <https://symfony.com/cloud>`_. Oferă tot ce avem nevoie și ajută la finanțarea dezvoltării Symfony.

.. index::
    single: Symfony CLI;project:init

Comanda ``symfony`` are suport integrat pentru SymfonyCloud. Hai să pornim un proiect SymfonyCloud:

.. code-block:: bash

    $ symfony project:init

Această comandă creează câteva fișiere necesare de SymfonyCloud, și anume ``.symfony/services.yaml``, ``.symfony/routes.yaml`` și ``.symfony.cloud.yaml``.

Adaugă-le în Git și salvează-le (git commit):

.. code-block:: bash

    $ git add .
    $ git commit -m"Add SymfonyCloud configuration"

.. note::

    Folosind comanda generală ``git add .`` funcționează, însă poate avea efecte adverse. Nu și în cazul nostru, deoarece în Symfony s-a creat un fișier ``.gitignore`` care exclude automat toate fișierele specifice mediului nostru, pe care nu dorim să le salvăm.

Punerea în funcțiune pe producție
------------------------------------

.. index::
    single: Symfony CLI;project:create
    single: Symfony CLI;deploy

Gata pentru lansare?

Creează un nou proiect SymfonyCloud:

.. code-block:: bash

    $ symfony project:create --title="Guestbook" --plan=development

Această comandă are mai multe roluri:

* Dacă e prima dată când rulezi această comandă, este necesară autentificarea cu credențialele SymfonyConnect.

* Îți va pregăti un nou proiect cu SymfonyCloud (la care vei avea 7 zile *gratuite* pentru orice proiect nou de dezvoltare)

Apoi, lansează:

.. code-block:: bash

    $ symfony deploy

Codul este lansat în producție prin salvarea în repozitoriul Git. După finalizarea execuției comenzii, proiectul va avea un nume de domeniu propriu pe care îl poți utiliza pentru a-l accesa pe internet.

.. index::
    single: Symfony CLI;open:remote

Verifică dacă totul a funcționat bine:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Ar trebui să vezi o eroare HTTP 404 pe prima pagină, dar dacă accesezi ``/images/under-construction.gif`` ar trebui să vezi creația noastră.

Reține că nu vei vedea frumoasa pagină implicită de la Symfony de pe SymfonyCloud. De ce? Vei afla în curând că Symfony acceptă mai multe medii lucru și SymfonyCloud a publicat automat codul folosind mediul de producție.

.. index::
    single: Symfony CLI;project:delete

.. tip::

    Dacă dorești să ștergi proiectul pe SymfonyCloud, folosește comanda ``project:delete``.

.. sidebar:: Mergând mai departe

    * `Serverul de rețete Flex pentru Symfony <https://flex.symfony.com/>`_, unde poți găsi toate rețetele disponibile pentru aplicațiile tale Symfony;

    * `Rețetele oficiale Symfony <https://github.com/symfony/recipes>`_ și pentru `rețetele cu contribuțiile comunității <https://github.com/symfony/recipes-contrib>`_, unde fiecare poate adăuga propriile rețete;

    * `Serverul Web Local Symfony <https://symfony.com/doc/current/setup/symfony_server.html>`_;

    * Documentația SymfonyCloud <https://symfony.com/doc/cloud>`_.
