Einführung einer Methodik
==========================

Beim Unterrichten geht es darum, das Gleiche immer wieder zu wiederholen. Das werde ich nicht tun, versprochen! Am Ende jedes Schrittes solltest Du ein Tänzchen hinlegen und Deine Arbeit speichern. Das ist wie ``Ctrl+S``, jedoch für eine Website.

Umsetzung einer Git-Strategie
-----------------------------

.. index::
    single: Git;add
    single: Git;commit

Vergiss nicht am Ende eines jeden Schrittes deine Änderungen zu committen:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Du kannst ohne Weiteres "alles" hinzufügen, da Symfony eine ``.gitignore`` Datei für Dich verwaltet. Und jedes Paket kann weitere Konfigurationen hinzufügen. Wirf einen Blick auf den aktuellen Inhalt:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,8

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Die seltsamen Zeichenketten sind Marker, die von Symfony Flex hinzugefügt wurden, damit es später weiß was es zu entfernen gilt, wenn Du dich entscheidest, eine Abhängigkeit zu deinstallieren. Ich habe ja gesagt, dass Dir Symfony die ganze mühsame Arbeit abnimmt.

Es ist sinnvoll dein Repository irgendwo auf einen Server zu pushen. GitHub, GitLab oder Bitbucket sind hierfür eine gute Wahl.

Falls Du auf der SymfonyCloud deployst, hast Du zwar bereits eine Kopie des Git-Repositorys. Darauf solltest Du Dich aber nicht verlassen. Es ist nur für das Deployment gedacht, nicht als Backup.

Kontinuierliches Deployment in die Produktivumgebung
----------------------------------------------------

.. index::
    single: Symfony CLI;deploy

Eine weitere gute Praxis sind häufige Deployments. Dabei ist ein gutes Tempo, jeweils am Ende eines jeden Schrittes zu deployen:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
