Crons uitvoeren
===============

.. index::
    single: Cron

Crons zijn nuttig om onderhoudstaken uit te voeren. In tegenstelling tot workers lopen ze voor een korte periode volgens een schema.

Reacties opschonen
------------------

Reacties die als spam zijn aangeduid of door de beheerder werden afgewezen, blijven in de database bewaard, omdat de beheerder de reacties mogelijk later nog wil kunnen bekijken. Maar waarschijnlijk moeten ze na een tijdje wel verwijderd worden. Het is waarschijnlijk voldoende om ze tot een week na hun creatie te behouden.

Maak een aantal utility-methods in de repository van reacties om afgekeurde reacties te vinden, ze te tellen en te verwijderen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -6,6 +6,7 @@ use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -18,6 +19,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -25,6 +28,29 @@ class CommentRepository extends ServiceEntityRepository
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
    +            ->setParameters([
    +                'state_rejected' => 'rejected',
    +                'state_spam' => 'spam',
    +                'date' => new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    Voor complexere queries is het soms handig om de gegenereerde SQL statements te bekijken (deze zijn te vinden in de logs en in de profiler voor webrequests).

Het gebruik van class-constants, containerparameters en omgevingsvariabelen
---------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 dagen? We hadden een ander nummer kunnen kiezen, misschien 10 of 20. Dit aantal kan in de loop van de tijd veranderen. We hebben besloten om dit op te slaan als een constante in de class, maar we hadden het ook op kunnen slaan als een parameter in de container, of we hadden het zelfs als omgevingsvariabele kunnen definiëren.

Hier zijn enkele vuistregels om te beslissen welke abstractie te gebruiken:

* Als de waarde gevoelig is (wachtwoorden, API-tokens, ....), gebruik dan de Symfony *secret storage* of een Vault;

* Als de waarde dynamisch is en je zou deze moeten kunnen aanpassen *zonder dat* je opnieuw hoeft te deployen, gebruik dan een *omgevingsvariabele*;

* Als de waarde per omgeving kan verschillen, gebruik dan een *containerparameter*;

* Voor al het andere, sla de waarde op in code, zoals in een *class constante*.

Een CLI-command maken
---------------------

Het verwijderen van de oude reacties is de perfecte taak voor een cronjob. Dit moet regelmatig gebeuren en een beetje vertraging heeft geen grote gevolgen.

Maak een CLI-command met de naam ``app:comment:cleanup`` door een ``src/Command/CommentCleanupCommand.php`` bestand aan te maken:

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

Alle commands van de applicatie zijn geregistreerd naast de Symfony ingebouwde commands en ze zijn allen toegankelijk via ``symfony console``. Aangezien het aantal beschikbare commands flink kan oplopen, moet je ze een namespace geven. Volgens conventie moeten de applicatie-commands zich in de ``app`` namespace bevinden. Voeg een willekeurig aantal subnamespaces toe door ze te scheiden met een dubbele punt ( ``:``).

Een command ontvangt de *input* (argumenten en opties die aan het command zijn doorgegeven) en je kunt de *output* gebruiken om naar de console te schrijven.

Ruim de database op door het command uit te voeren:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Het instellen van een cron op Platform.sh
-----------------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

Een van de leuke dingen van Platform.sh is dat het grootste deel van de configuratie in één bestand is opgeslagen: ``.platform.app.yaml``. De webcontainer, de workers en de cronjobs worden samen beschreven om het onderhoud te vergemakkelijken:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -61,6 +61,14 @@ crons:
             spec: '50 23 * * *'
             cmd: if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then croncape php-security-checker; fi

    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messenger:
             commands:

De ``crons`` sectie definieert alle cronjobs. Elke cron wordt uitgevoerd volgens een ``spec`` schema.

Het ``croncape`` hulpprogramma monitort de uitvoering van het command en stuurt een e-mail naar de adressen die zijn gedefinieerd in de ``MAILTO`` omgevingsvariabele als het command een andere exitcode dan ``0`` heeft.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Configureer de ``MAILTO`` omgevingsvariabele:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Merk op dat er in alle Platform.sh branches crons opgezet zijn. Als je niet wilt dat sommige crons op niet-productie omgevingen worden uitgevoerd, controleer dan de ``$PLATFORM_ENVIRONMENT_TYPE`` omgevingsvariabele:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Verder gaan

    * `Cron/crontab syntaxis`_;

    * `Croncape repository`_;

    * `Symfony Console commands`_;

    * De `Symfony Console Cheat Sheet`_.

.. _`Cron/crontab syntaxis`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console commands`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
