Prezentarea proiectului
=======================

Trebuie să găsim un proiect la care să lucrăm, iar asta este o provocare, deoarece trebuie să găsim un proiect suficient de mare pentru a acoperi Symfony în profunzime, dar în același timp, ar trebui să fie accesibil. Nu vreau să te plictisesc implementând funcționalități similare de mai multe ori.

Dezvăluirea proiectului
------------------------

Deoarece cartea a fost lansată în timpul SymfonyCon Amsterdam, ar fi frumos dacă proiectul este legat cumva de Symfony și conferințe. Ce zici de un `guestbook <https://en.wikipedia.org/wiki/Guestbook>`_? „Un livre d’or”, cum spunem în franceză. Îmi place senzația de modă veche și depășită de a dezvolta o carte de oaspeți în 2019!

Proiectul colectează recenziile participanților la conferințe: o listă de conferințe pe pagina principală, o pagină pentru fiecare conferință, plină de comentarii frumoase. Un comentariu este format dintr-un text scurt și o fotografie opțională realizată în timpul conferinței. Presupun că tocmai am notat toate specificațiile de care avem nevoie pentru a începe.

*Proiectul* va conține mai multe *aplicații*. O aplicație web tradițională cu un frontend HTML, un API și un SPA (single page application) pentru telefoane mobile. Sună bine?

Învățare prin practică
--------------------------

A învăța înseamnă a practica. Punct. A citi o carte despre Symfony sună bine. Dezvoltarea unei aplicații software pe calculatorul tău în timp ce citești o carte despre Symfony sună și mai bine. Această carte este foarte specială, deoarece totul a fost pregătit să îți permită să o urmezi, dezvoltând, și să fii sigur că vei avea aceleași rezultate pe care le-am avut eu pe mașina mea locală atunci când am scris aplicația inițială.

Cartea conține tot codul pe care trebuie să-l scrii și toate comenzile pe care trebuie să le execuți pentru a obține rezultatul final. Nu lipsește nimic. Toate comenzile sunt menționate. Acest lucru este posibil deoarece aplicațiile Symfony moderne au foarte puțin cod de structură. Cea mai mare parte a codului pe care îl vom scrie împreună descrie *logica* de business. Orice altceva este automatizat sau generat în mod automat pentru noi.

Analizând schema finală a arhitecturii
----------------------------------------

Chiar dacă ideea de proiect pare simplă, nu vom construi un proiect de tip „Hello World”. Nu vom folosi doar PHP și o bază de date.

Scopul este de a crea un proiect la fel de complex ca unul pe care îl poți găsi în viața de zi cu zi. Vrei o dovadă? Aruncă o privire asupra infrastructurii finale a proiectului:

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

Unul dintre avantajele mari ale utilizării unui framework este cantitatea mică de cod necesară dezvoltării unui astfel de proiect:

* 20 de clase PHP în directorul ``src/`` pentru site-ul web;

* 550 de linii de cod PHP pentru logică, raportat de `PHPLOC <https://github.com/sebastianbergmann/phploc>`_;

* 40 de linii de ajustări de configurare în 3 fișiere (prin adnotări și YAML), în principal pentru a configura aspectul panoului de administrare;

* 20 de linii de configurare a infrastructurii de dezvoltare (Docker);

* 100 de linii de configurare a infrastructurii de producție (SymfonyCloud);

* 5 variabile explicite de mediu.

Ești gata de provocare?

Obținerea codului sursă a proiectului
---------------------------------------

Dacă aș fi continuat cu moda veche, aș fi scris un CD cu codul sursă, nu? Dar, nu mai bine folosim un repozitoriu Git?

.. index::
    single: Project;Git Repository
    single: Git;clone

Clonează `repozitoriul cărții de oaspeți <https://github.com/the-fast-track/book-5.2-2>`_ undeva pe calculatorul tău:

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.2-2 --book guestbook

Acest repozitoriu conține tot codul menționat în carte.

Reține că folosim ``symfony new`` în loc de ``git clone``, deoarece comanda îndeplinește mai mult decât doar clonarea codului (găzduit pe Github în cadrul organizației ``the-fast-track '': ``https: // github.com / the-fast-track / 5.2-2``). De asemenea, pornește serverul web, containerele, asigură structura bazei de date, încărcă datele de testare... După executarea comenzii, site-ul web ar trebui să fie accesibil, gata de utilizare.

Codul corespunde sută la sută cu codul din carte (utilizează URL-ul exact al repozitoriului menționat mai sus). Încercarea de a sincroniza manual modificările din carte cu codul sursă din repozitoriu este aproape imposibilă. Am încercat în trecut. Am eșuat. Este pur și simplu imposibil. Mai ales pentru cărți precum cele pe care le scriu: cărți care vă spun o poveste despre dezvoltarea unui site web. Deoarece fiecare capitol depinde de cele anterioare, o schimbare poate avea consecințe în toate capitolele următoare.

Vestea bună este că repozitoriul Git pentru această carte este *generat automat* din conținutul cărții. Ai citit corect. Mie îmi place să automatizez totul, așa că există un script al cărui rol este să scaneze cartea și să creeze codul sursă în Git. Iar asta are un avantaj suplimentar: la actualizarea cărții, scriptul va da eroare dacă modificările nu corespund sau dacă uit să actualizez vreo instrucțiune. Asta înseamnă BDD: Book Driven Development!

Navigând codul sursă
----------------------

Și mai bine, în Git nu este doar versiunea finală a codului din ``master``. Scriptul execută fiecare acțiune explicată în carte și salvează codul la sfârșitul fiecărei secțiuni. De asemenea, etichetează fiecare pas pentru a ușura navigarea prin cod. Frumos, nu?

.. index::
    single: Git;checkout

Dacă ești leneș, poți obține codul de la sfârșitul unui pas dacă folosești eticheta potrivită (git tag). De exemplu, dacă la sfârșitul pasului 10 dorești să vezi și să testezi codul sursă, execută următoarele:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Ca și la clonarea din Git, nu folosim ``git checkout``, ci ``symfony book:checkout``. Comanda se asigură că, indiferent de starea în care te afli, rămâi cu un site web funcțional pentru pasul pe care îl soliciți. **Fii atent! Toate datele, codul și containerele create anterior sunt șterse prin această operațiune.**

Poți naviga la orice pas pentru a obține codul aferent:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

Dar, îți recomand cu încredere să scrii singur codul. Dacă te-ai blocat însă, poți să compari mereu ceea ce ai cu conținutul cărții.

.. index::
    single: Git;diff

Nu ești sigur că ai făcut totul corect în pasul 10.2? Verifică diferența:

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Vrei să știi când a fost creat sau modificat un fișier?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

De asemenea poți vedea diferențe de cod (git diff), etichete (git tag) și commit-uri direct pe GitHub. În felul acesta poți copia codul dacă citești cartea tipărită!
