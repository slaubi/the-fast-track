Aufgaben planen
===============

.. index::
    single: Cron

Manche Wartungsaufgaben müssen nach einem Zeitplan laufen. Im Gegensatz zu Workern, die kontinuierlich laufen, werden geplante Aufgaben periodisch für einen kurzen Zeitraum ausgeführt.

Kommentare bereinigen
---------------------

Kommentare, die als Spam markiert oder von Administrator*innen abgelehnt werden, werden in der Datenbank gespeichert, damit Administrator*innen sie noch eine Weile begutachten können. Sie sollten jedoch nach einiger Zeit entfernt werden. Es reicht vermutlich aus, sie für eine Woche zu behalten.

Erstelle ein paar Hilfsmethoden im Kommentar-Repository, um abgelehnte Kommentare zu finden, zu zählen und zu löschen:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -5,7 +5,9 @@ namespace App\Repository;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    +use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -13,6 +15,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const COMMENTS_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -20,6 +24,27 @@ class CommentRepository extends ServiceEntityRepository
             parent::__construct($registry, Comment::class);
         }

    +    public function countOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->select('COUNT(c.id)')->getQuery()->getSingleScalarResult();
    +    }
    +
    +    public function deleteOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->delete()->getQuery()->execute();
    +    }
    +
    +    private function getOldRejectedQueryBuilder(): QueryBuilder
    +    {
    +        return $this->createQueryBuilder('c')
    +            ->andWhere('c.state = :state_rejected or c.state = :state_spam')
    +            ->andWhere('c.createdAt < :date')
    +            ->setParameter('state_rejected', 'rejected')
    +            ->setParameter('state_spam', 'spam')
    +            ->setParameter('date', new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'))
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    Bei komplexeren Abfragen ist es manchmal sinnvoll, sich die erzeugten SQL-Anweisungen anzusehen (sie befinden sich in den Logs und im Profiler für Web-Anfragen).

Klassen-Konstanten, Container-Parameter und Environment-Variablen verwenden
---------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 Tage? Wir hätten eine andere Zahl wählen können, vielleicht 10 oder 20. Diese Zahl kann sich im Laufe der Zeit ändern. Wir haben beschlossen, sie als Konstante in der Klasse zu speichern, aber wir könnten sie auch als Parameter im Container speichern, oder sogar als Environment-Variable definieren.

Hier sind einige Faustregeln, um zu entscheiden, welche Abstraktion verwendet werden soll:

* Wenn der Wert sensibel ist (Passwörter, API-Token,...), verwende den Symfony *Secret Storage* oder einen Vault;

* Wenn der Wert dynamisch ist und Du ihn ändern können musst, *ohne*  erneut zu deployen, verwende eine *Environment-Variable*;

* Wenn der Wert zwischen den Environments unterschiedlich sein kann, verwende einen *Container-Parameter*;

* Für alles andere setzt Du den Wert im Code, zum Beispiel in einer *Klassenkonstanten*.

Ein CLI-Befehl erstellen
------------------------

Das Entfernen der alten Kommentare ist die perfekte Aufgabe für einen Cron-Job. Es sollte regelmäßig durchgeführt werden, und eine kleine Verzögerung hat keine größeren Auswirkungen.

Erstelle einen CLI-Befehl ``app:comment:cleanup``, indem Du eine ``src/Command/CommentCleanupCommand.php``-Datei anlegst:

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Attribute\Option;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand
    {
        public function __invoke(
            SymfonyStyle $io,
            CommentRepository $commentRepository,
            #[Option(description: 'Dry run')]
            bool $dryRun = false,
        ): int {
            if ($dryRun) {
                $io->note('Dry mode enabled');

                $count = $commentRepository->countOldRejected();
            } else {
                $count = $commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Alle Anwendungsbefehle werden parallel zu den in Symfony eingebauten Befehlen registriert und sind über  ``symfony console`` erreichbar. Da die Anzahl der verfügbaren Befehle groß sein kann, solltest Du ihnen einen Namespace geben. Nach Konvention sollten die Anwendungsbefehle unter dem ``app``-Namespace abgelegt werden. Du kannst beliebig viele Sub-Namespaces hinzufügen, indem Du diese durch einen Doppelpunkt (``:``) trennst.

Ein Befehl deklariert seine *Argumente* und *Optionen* mit den Attributen ``#[Argument]`` und ``#[Option]`` an den Parametern von ``__invoke()`` (der Parameter ``$dryRun`` wird zur Option ``--dry-run``). Symfony injiziert die anderen Parameter anhand ihres Typs: ``SymfonyStyle``, um schön formatierte Ausgaben in die Konsole zu schreiben, und jeden beliebigen Service, wie das Kommentar-Repository, genauso wie bei Controller-Argumenten.

Bereinige die Datenbank, indem Du diesen Befehl ausführst:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Den Befehl planen
-----------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Den Befehl von Hand auszuführen funktioniert, aber er sollte jede Nacht laufen. Die Symfony-Scheduler-Komponente erzeugt Messages nach einem Zeitplan; sie werden dann von einem Worker konsumiert, wie jede andere Messenger-Message.

Füge die Scheduler-Komponente hinzu, zusammen mit der Bibliothek, die Cron-Ausdrücke parst:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Plane den Befehl mit dem ``#[AsCronTask]``-Attribut:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/CommentCleanupCommand.php
    +++ w/src/Command/CommentCleanupCommand.php
    @@ -7,8 +7,10 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Attribute\Option;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Style\SymfonyStyle;
    +use Symfony\Component\Scheduler\Attribute\AsCronTask;

     #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    +#[AsCronTask('50 23 * * *')]
     class CommentCleanupCommand
     {
         public function __invoke(

Das Attribut registriert den Befehl im Standard-*Schedule* mit einem Cron-Ausdruck: jede Nacht um 23:50 Uhr (UTC). Überprüfe es:

.. code-block:: terminal

    $ symfony console debug:scheduler

Ein Schedule wird als gewöhnlicher Messenger-Transport mit seinem Namen bereitgestellt; konsumiere ihn wie jeden anderen Transport:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Den Schedule deployen
---------------------

.. index::
    single: Upsun;Workers

Auf Upsun konsumiert der Worker nur den ``async``-Transport. Lass ihn auch den Schedule konsumieren:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -87,4 +87,4 @@ applications:
             messenger:
                 commands:
                     # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
    -                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async
    +                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async scheduler_default

Mehr braucht es nicht: kein Crontab, kein zusätzlicher Prozess; der Zeitplan lebt im PHP-Code, direkt neben der Aufgabe, die er auslöst, und wird wie der Rest der Anwendung deployt und versioniert.

Was ist mit System-Crons?
-------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun unterstützt auch Cron-Jobs auf Betriebssystemebene, die in ``.upsun/config.yaml`` neben dem Web-Container und den Workern beschrieben werden; die Standardkonfiguration definiert bereits einen, der abgelaufene PHP-Sessions bereinigt. System-Crons eignen sich gut für Aufgaben, die nicht in PHP implementiert sind.

Das vom Standard-Cron verwendete ``croncape``-Dienstprogramm überwacht die Ausführung des Befehls und sendet eine E-Mail an die Adressen, die in der ``MAILTO``-Environment-Variable definiert sind, falls der Befehl einen anderen Exit-Code als ``0`` zurückgibt:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Beachte, dass Cron-Jobs auf allen Upsun-Branches eingerichtet werden. Wenn Du manche davon nicht in Nicht-Produktionsumgebungen ausführen möchtest, überprüfe die ``$PLATFORM_ENVIRONMENT_TYPE``-Environment-Variable:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Weiterführendes

    * Die `Scheduler-Komponenten-Dokumentation`_;

    * `Cron-Jobs/Crontab-Syntax`_;

    * `Croncape-Repository`_;

    * `Symfony Console Befehle`_;

    * Der `Symfony Console Cheat Sheet`_.

.. _`Scheduler-Komponenten-Dokumentation`: https://symfony.com/doc/current/scheduler.html
.. _`Cron-Jobs/Crontab-Syntax`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape-Repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console Befehle`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
