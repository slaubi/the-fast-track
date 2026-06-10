Cron-Jobs ausführen
====================

.. index::
    single: Cron

Cron-Jobs sind nützlich, um Wartungsarbeiten durchzuführen. Im Gegensatz zu Workern laufen sie gemäß einem Zeitplan für einen kurzen Zeitraum.

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
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand extends Command
    {
        public function __construct(
            private CommentRepository $commentRepository,
        ) {
            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->addOption('dry-run', null, InputOption::VALUE_NONE, 'Dry run')
            ;
        }

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $io = new SymfonyStyle($input, $output);

            if ($input->getOption('dry-run')) {
                $io->note('Dry mode enabled');

                $count = $this->commentRepository->countOldRejected();
            } else {
                $count = $this->commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Alle Anwendungsbefehle werden parallel zu den in Symfony eingebauten Befehlen registriert und sind über  ``symfony console`` erreichbar. Da die Anzahl der verfügbaren Befehle groß sein kann, solltest Du ihnen einen Namespace geben. Nach Konvention sollten die Anwendungsbefehle unter dem ``app``-Namespace abgelegt werden. Du kannst beliebig viele Sub-Namespaces hinzufügen, indem Du diese durch einen Doppelpunkt (``:``) trennst.

Ein Befehl erhält die *Eingabe* (Input; Argumente und Optionen, die an den Befehl übergeben wurden) und Du kannst die *Ausgabe* (Output) verwenden, um Information in der Konsole auszugeben.

Bereinige die Datenbank, indem Du diesen Befehl ausführst:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Einen Cron-Job bei Upsun einrichten
-----------------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Das Schöne an Upsun ist, dass der Großteil der Konfiguration in einer Datei gespeichert ist: ``.upsun/config.yaml``. Der Webcontainer, die Worker und die Cron-Jobs werden gemeinsam definiert, um die Wartung zu erleichtern:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -83,5 +83,13 @@ applications:
                     spec: '17,47 * * * *'
                     commands:
                         start: croncape php-session-clean
    +            comment_cleanup:
    +                # Cleanup every night at 11.50 pm (UTC).
    +                spec: '50 23 * * *'
    +                commands:
    +                    start: |
    +                        if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
    +                            croncape symfony console app:comment:cleanup
    +                        fi

             workers:

Der ``crons``-Abschnitt definiert alle Cron-Jobs. Jeder Cron-Job läuft nach einem ``spec``-Zeitplan.

Das ``croncape``-Dienstprogramm überwacht die Ausführung des Befehls und sendet eine E-Mail an die in der Environment-Variable ``MAILTO`` definierten Adressen, wenn der Befehl einen anderen Exit-Code als ``0`` hat.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Konfiguriere die Environment-Variable ``MAILTO``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Beachte, dass Cron-Jobs auf allen Upsun-Branches eingerichtet sind. Überprüfe die Environment-Variable ``$PLATFORM_ENVIRONMENT_TYPE`` wenn Du keine Cron-Jobs in Nicht-Production-Environments ausführen möchtest:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Weiterführendes

    * `Cron-Jobs/Crontab-Syntax`_;

    * `Croncape-Repository`_;

    * `Symfony Console Befehle`_;

    * Der `Symfony Console Cheat Sheet`_.

.. _`Cron-Jobs/Crontab-Syntax`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape-Repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console Befehle`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
