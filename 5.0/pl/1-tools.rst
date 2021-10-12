Przygotowanie środowiska pracy
===============================

Zanim rozpoczniemy pracę nad projektem, musimy upewnić się, że mamy właściwe środowisko pracy. To bardzo ważne. Narzędzia programistyczne, którymi dzisiaj dysponujemy, bardzo różnią się od tych, których używaliśmy 10 lat temu. Przeszły pozytywną zmianę, szkoda byłoby zatem z nich nie skorzystać. Odpowiednio dobrane narzędzia bardzo ułatwią nam pracę.

Proszę, nie pomijaj tego kroku. Przeczytaj przynajmniej ostatnią sekcję o Symfony CLI.

Komputer
--------

Potrzebujesz komputera. Dobra wiadomość jest taka, że może na nim działać dowolny, popularny system operacyjny: macOS, Windows lub Linux. Symfony i wszystkie narzędzia, których będziemy używać, są kompatybilne z każdym z nich.

Podjęte decyzje projektowe
---------------------------

Chciałbym szybko przejść przez wybór dostępnych opcji. Na potrzeby tej książki podjąłem kilka decyzji projektowych kierując się swoimi doświadczeniami.

`PostgreSQL`_ będzie naszym wyborem jako silnik bazodanowy.

`RabbitMQ`_ zwycięża wśród systemów kolejkowania.

IDE
---

.. index:: IDE

Możesz użyć notatnika, jeśli chcesz, aczkolwiek nie polecam.

Miałem okazję pracować w Textmate - nigdy więcej. Komfort korzystania z "prawdziwego" IDE jest bezcenny. Autouzupełnianie, automatyczne dodawanie i sortowanie instrukcji ``use``, czy płynne przechodzenie między kolejnymi plikami projektu, to tylko kilka z wielu funkcji, które zwiększą Twoją produktywność oraz wygodę pracy.

Osobiście polecam skorzystać z jednego z dwóch IDE: `Visual Studio Code`_ lub `PhpStorm`_. Pierwszy z nich jest darmowy, drugi już nie, ale posiada za to dużo lepszą integrację z Symfony (dzięki wtyczce `Symfony Support Plugin`_) - wybór zostawiam Tobie. Pewnie chcielibyście wiedzieć, z którego IDE osobiście korzystam? Ta książka została napisana z wykorzystaniem Visual Studio Code.

Terminal
--------

.. index:: Terminal

Bardzo często będziemy przełączać się między IDE a terminalem. Możesz używać terminala wbudowanego w IDE, ale ja wolę używać systemowego, aby mieć więcej miejsca.

Linux posiada wbudowany ``Terminal``. Na MacOS użyj `iTerm2`_. W systemie Windows `Hyper`_ jest dobrym wyborem.

Git
---

.. index:: Git

W mojej ostatniej książce polecałem Subversion jako system kontroli wersji. Wygląda na to, że teraz wszyscy używają `Gita`_.

Jeśli używasz systemu Windows, zainstaluj `Git BASH`_.

Upewnij się, że wiesz, jak wykonywać podstawowe operacje, takie jak: ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``, ...

PHP
---

.. index:: PHP

Będziemy używać Dockera dla usług, lecz PHP lubię mieć zainstalowany na lokalnym komputerze dla wydajności, stabilności i prostoty. Nazywaj mnie staromodnym, jeśli chcesz, ale połączenie lokalnego PHP i Dockera jest dla mnie idealne.

Jeśli możesz, użyj PHP 7.3 lub 7.4 w zależności od tego kiedy czytasz tę książkę. Sprawdź, czy :index:`następujące rozszerzenia PHP <PHP extensions>` są zainstalowane. Jeżeli nie, zainstaluj je teraz: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``. Opcjonalnie możesz zainstalować również  ``redis`` i ``curl``.

Możesz sprawdzić aktualnie włączone rozszerzenia poprzez ``php -m``.

Będziemy także potrzebować ``php-fpm``, jeśli jest dostępny na Twojej platformie. Możesz również wykorzystać ``php-cgi``.

Composer
--------

.. index:: Composer

Zarządzanie zależnościami jest obecnie niezbędne w projekcie Symfony. Pobierz najnowszą wersję `Composer <https://getcomposer.org/>`_, narzędzia do zarządzania pakietami dla PHP.

Jeśli nie znasz narzędzia Composer, poświęć chwilę na zapoznanie się z nim.

.. tip::

    Nie musisz wpisywać pełnych nazw poleceń: ``composer req`` robi to samo co ``composer require``, użyj ``composer rem`` zamiast ``composer remove``, ....

Docker i Docker Compose
-----------------------

.. index:: Docker,Docker Compose

Usługi będą zarządzane przez platformę Docker oraz narzędzie Docker Compose. `Zainstaluj je <https://docs.docker.com/install/>`_, a następnie uruchom Dockera. Jeśli nie znasz Dockera, poznaj go. Nie ma powodów do obaw, cały proces będzie bardzo prosty. Bez dziwnych konfiguracji, bez skomplikowanych ustawień.

Symfony CLI
-----------

.. index:: Symfony CLI

Wreszcie, co nie mniej ważne, wykorzystamy ``symfony`` CLI, aby zwiększyć naszą produktywność. Od uruchomienia lokalnego serwera WWW, po pełną integrację Dockera i obsługę SymfonyCloud - oszczędzimy mnóstwo czasu.

Zainstaluj `Symfony CLI <https://symfony.com/download>`_ i dodaj jego ścieżkę do zmiennej ``$PATH``. Załóż konto `SymfonyConnect <https://connect.symfony.com/>`_, jeśli jeszcze go nie masz, i zaloguj się przez ``symfony login``.

Aby korzystać z HTTPS lokalnie, musimy również `zainstalować CA <https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls>`_, aby włączyć obsługę TLS. Wykonaj następujące polecenie:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Sprawdź, czy Twój komputer spełnia wszystkie niezbędne wymagania, wykonując następujące polecenie:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

Jeśli masz ochotę, możesz również uruchomić `serwer proxy Symfony <https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy>`_. Jest to opcjonalne, ale pozwala na uzyskanie nazwy domeny lokalnej kończącej się na ``.wip`` w Twoim projekcie.

Gdy wykonujemy polecenie w terminalu, prawie zawsze będziemy je poprzedzać słowem ``symfony``, jak na przykład ``symfony composer`` zamiast samego ``composer``, lub ``symfony console`` zamiast ``./bin/console``.

Głównym powodem jest to, że narzędzie Symfony CLI automatycznie ustawia niektóre zmienne środowiskowe w oparciu o usługi działające na Twojej maszynie poprzez Dockera. Te zmienne środowiskowe są dostępne dla żądań (ang. requests) HTTP, ponieważ lokalny serwer WWW wstrzykuje je automatycznie. Tak więc, użycie ``symfony`` w konsoli gwarantuje, że działanie aplikacji będzie takie samo w linii komend jak na stronie WWW.

Ponadto, Symfony CLI automatycznie wybiera "najlepszą" możliwą wersję PHP dla projektu.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Gita`: https://git-scm.com/
.. _`Git BASH`: https://gitforwindows.org/
