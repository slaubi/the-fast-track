Taken plannen
=============

.. index::
    single: Cron

Sommige onderhoudstaken moeten volgens een schema draaien. In tegenstelling tot workers, die continu draaien, draaien geplande taken periodiek voor een korte periode.

Reacties opschonen
------------------

Reacties die als spam zijn aangeduid of door de beheerder werden afgewezen, blijven in de database bewaard, omdat de beheerder de reacties mogelijk later nog wil kunnen bekijken. Maar waarschijnlijk moeten ze na een tijdje wel verwijderd worden. Het is waarschijnlijk voldoende om ze tot een week na hun creatie te behouden.

Maak een aantal utility-methods in de repository van reacties om afgekeurde reacties te vinden, ze te tellen en te verwijderen:

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

Alle commands van de applicatie zijn geregistreerd naast de Symfony ingebouwde commands en ze zijn allen toegankelijk via ``symfony console``. Aangezien het aantal beschikbare commands flink kan oplopen, moet je ze een namespace geven. Volgens conventie moeten de applicatie-commands zich in de ``app`` namespace bevinden. Voeg een willekeurig aantal subnamespaces toe door ze te scheiden met een dubbele punt ( ``:``).

Een command declareert zijn *argumenten* en *opties* met de ``#[Argument]`` en ``#[Option]`` attributen op de parameters van ``__invoke()`` (de ``$dryRun`` parameter wordt de ``--dry-run`` optie). Symfony injecteert de overige parameters op basis van hun type: ``SymfonyStyle`` om netjes opgemaakte output naar de console te schrijven, en elke service, zoals de repository van reacties, op dezelfde manier als bij controller-argumenten.

Ruim de database op door het command uit te voeren:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Het command plannen
-------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Het command handmatig uitvoeren werkt, maar het zou elke nacht moeten draaien. De Symfony Scheduler-component genereert berichten volgens een schema; deze worden vervolgens door een worker geconsumeerd, net als alle andere Messenger-berichten.

Voeg de Scheduler-component toe, samen met de library die cron-expressies parseert:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Plan het command met het ``#[AsCronTask]`` attribuut:

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

Het attribuut registreert het command op het standaard-*schema* met een cron-expressie: elke nacht om 23.50 uur (UTC). Controleer het:

.. code-block:: terminal

    $ symfony console debug:scheduler

Een schema wordt blootgesteld als een gewone Messenger-transport met dezelfde naam; consumeer het zoals elke andere transport:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Het schema deployen
-------------------

.. index::
    single: Upsun;Workers

Op Upsun consumeert de worker alleen de ``async`` transport. Laat hem ook het schema consumeren:

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

Meer is er niet nodig: geen crontab, geen extra proces; het schema leeft in de PHP-code, naast de taak die het activeert, en het wordt gedeployd en geversioneerd zoals de rest van de applicatie.

Hoe zit het met systeemcrons?
-----------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun ondersteunt ook cronjobs op OS-niveau, beschreven in ``.upsun/config.yaml`` naast de webcontainer en de workers; de standaardconfiguratie definieert er al een die verlopen PHP-sessies opschoont. Systeemcrons passen goed bij taken die niet in PHP zijn geïmplementeerd.

Het ``croncape`` hulpprogramma dat door de standaardcron wordt gebruikt, monitort de uitvoering van het command en stuurt een e-mail naar de adressen die zijn gedefinieerd in de ``MAILTO`` omgevingsvariabele als het command een andere exitcode dan ``0`` teruggeeft:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Merk op dat er in alle Upsun branches crons opgezet zijn. Als je niet wilt dat sommige crons op niet-productie omgevingen worden uitgevoerd, controleer dan de ``$PLATFORM_ENVIRONMENT_TYPE`` omgevingsvariabele:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Verder gaan

    * De `Scheduler component docs`_;

    * `Cron/crontab syntaxis`_;

    * `Croncape repository`_;

    * `Symfony Console commands`_;

    * De `Symfony Console Cheat Sheet`_.

.. _`Scheduler component docs`: https://symfony.com/doc/current/scheduler.html
.. _`Cron/crontab syntaxis`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape repository`: https://github.com/symfonycorp/croncape
.. _`Symfony Console commands`: https://symfony.com/doc/current/console.html
.. _`Symfony Console Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
