Pianificare le attività
=======================

.. index::
    single: Cron

Alcune attività di manutenzione devono essere eseguite secondo una pianificazione. A differenza dei worker, che girano in continuazione, le attività pianificate vengono eseguite periodicamente per un breve periodo di tempo.

Pulire i commenti
-----------------

I commenti contrassegnati come spam o rifiutati dall'amministratore sono conservati nel database, in quanto l'amministratore potrebbe volerli ispezionare per un po' di tempo. Ma probabilmente dovrebbero essere rimossi in un secondo momento. Probabilmente è sufficiente conservarli per una settimana.

Creare alcuni metodi nel repository dei commenti: per trovare i commenti rifiutati, per contarli e per cancellarli:

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

    Per le query più complesse, a volte è utile dare un'occhiata alle istruzioni SQL generate (si possono trovare nei log e nel profiler per le richieste web).

Utilizzo delle costanti di classe, dei parametri del container e delle variabili d'ambiente
-------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

Sette giorni? Avremmo potuto scegliere un altro numero, forse dieci o venti. Questo numero potrebbe evolvere nel tempo. Abbiamo deciso di memorizzarlo come costante di classe, ma avremmo potuto memorizzarlo come parametro nel container o ancora come variabile d'ambiente.

Ecco alcune regole empiriche per decidere quale astrazione usare:

* Se il valore deve essere mantenuto segreto (password, token API, ...), usare il portachiavi di Symfony o un sistema esterno di portachiavi;

* Se il valore è dinamico e lo si può cambiare *senza* dover rifare un deploy, usare una *variabile d'ambiente*;

* Se il valore può essere diverso da un ambiente all'altro, usare un *parametro del container*;

* Per tutto il resto, memorizzare il valore nel codice, come *costante di classe*.

Creazione di un comando CLI
---------------------------

La rimozione dei vecchi commenti è il compito perfetto per un cron. Andrebbe fatta su base regolare e un piccolo ritardo non ha un impatto significativo.

Creare un comando CLI denominato ``app:comment:cleanup``, creando un file ``src/Command/CommentCleanupCommand.php``:

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

Tutti i comandi dell'applicazione sono registrati insieme a quelli integrati in Symfony e sono tutti accessibili tramite ``symfony console``. Poiché il numero di comandi disponibili potrebbe essere elevato, dovremmo raggrupparli per nome. Per convenzione, i comandi dell'applicazione vengono raggruppati sotto il namespace ``app``. Si possono aggiungere ulteriori namespace, separati con i due punti (``:``).

Un comando dichiara i suoi *parametri* e le sue *opzioni* con gli attributi ``#[Argument]`` e ``#[Option]`` sui parametri di ``__invoke()`` (il parametro ``$dryRun`` diventa l'opzione ``--dry-run``). Symfony inietta gli altri parametri in base al loro tipo: ``SymfonyStyle`` per scrivere in console un output ben formattato, e qualsiasi servizio, come il repository dei commenti, allo stesso modo degli argomenti dei controller.

Pulire il database eseguendo il comando:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Pianificare il comando
----------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Eseguire il comando a mano funziona, ma dovrebbe girare ogni notte. Il componente Symfony Scheduler genera messaggi secondo una pianificazione; vengono poi consumati da un worker, come qualsiasi altro messaggio Messenger.

Aggiungere il componente Scheduler, insieme alla libreria che analizza le espressioni cron:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Pianificare il comando con l'attributo ``#[AsCronTask]``:

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

L'attributo registra il comando sulla *pianificazione* (schedule) predefinita con un'espressione cron: ogni notte alle 23:50 (UTC). Verificarlo:

.. code-block:: terminal

    $ symfony console debug:scheduler

Una pianificazione viene esposta come un normale trasporto Messenger che ne porta il nome; consumarla come qualsiasi altro trasporto:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Distribuire la pianificazione
-----------------------------

.. index::
    single: Upsun;Workers

Su Upsun, il worker consuma solo il trasporto ``async``. Fargli consumare anche la pianificazione:

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

Non serve altro: nessun crontab, nessun processo aggiuntivo; la pianificazione vive nel codice PHP, accanto all'attività che attiva, e viene distribuita e versionata come il resto dell'applicazione.

E i cron di sistema?
--------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun supporta anche i cron a livello di sistema operativo, descritti in ``.upsun/config.yaml`` accanto al container web e ai worker; la configurazione predefinita ne definisce già uno che ripulisce le sessioni PHP scadute. I cron di sistema sono adatti alle attività che non sono implementate in PHP.

L'utility ``croncape`` usata dal cron predefinito monitora l'esecuzione del comando e invia un'email agli indirizzi definiti nella variabile d'ambiente ``MAILTO``, se il comando restituisce un codice di uscita diverso da ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Si noti che i cron vengono impostati su tutti i rami di Upsun. Se non si vuole eseguirne alcuni in ambienti diversi dalla produzione, verificare la variabile d'ambiente ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Andare oltre

    * La `documentazione del componente Scheduler`_;

    * `Sintassi di cron e crontab`_;

    * `Repository di Croncape`_;

    * `Comandi della console di Symfony`_;

    * `Cheat sheet della console di Symfony`_.

.. _`documentazione del componente Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`Sintassi di cron e crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Repository di Croncape`: https://github.com/symfonycorp/croncape
.. _`Comandi della console di Symfony`: https://symfony.com/doc/current/console.html
.. _`Cheat sheet della console di Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
