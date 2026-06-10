Einführung einer Methodik
==========================

Beim Unterrichten geht es darum, das Gleiche immer wieder zu wiederholen. Das werde ich nicht tun, versprochen! Am Ende jedes Schrittes solltest Du ein Tänzchen hinlegen und Deine Arbeit speichern. Das ist wie ``Ctrl+S``, jedoch für eine Website.

Umsetzung einer Git-Strategie
-----------------------------

.. index::
    single: Git;add
    single: Git;commit

Vergiss nicht am Ende eines jeden Schrittes deine Änderungen zu committen:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Du kannst ohne Weiteres "alles" hinzufügen, da Symfony eine ``.gitignore`` Datei für Dich verwaltet. Und jedes Paket kann weitere Konfigurationen hinzufügen. Wirf einen Blick auf den aktuellen Inhalt:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Die seltsamen Zeichenketten sind Marker, die von Symfony Flex hinzugefügt wurden, damit es später weiß was es zu entfernen gilt, wenn Du dich entscheidest, eine Abhängigkeit zu deinstallieren. Ich habe ja gesagt, dass Dir Symfony die ganze mühsame Arbeit abnimmt.

Es ist sinnvoll dein Repository irgendwo auf einen Server zu pushen. GitHub, GitLab oder Bitbucket sind hierfür eine gute Wahl.

.. note::

    If you are deploying on Platform.sh, you already have a copy of the Git repository as Platform.sh uses Git behind the scenes when you are using ``cloud:push``. But you should not rely on the Platform.sh Git repository. It is only for deployment usage. It is not a backup.

Kontinuierliches Deployment in die Produktivumgebung
----------------------------------------------------

.. index::
    single: Symfony CLI;cloud:push

Eine weitere gute Praxis sind häufige Deployments. Dabei ist ein gutes Tempo, jeweils am Ende eines jeden Schrittes zu deployen:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
