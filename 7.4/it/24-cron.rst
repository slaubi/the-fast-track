Eseguire Cron
=============

.. index::
    single: Cron

I cron sono utili per svolgere attività di manutenzione. A differenza dei worker, hanno un orario di esecuzione predefinito, e vengono eseguiti per un breve periodo di tempo.

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

Tutti i comandi dell'applicazione sono registrati insieme a quelli integrati in Symfony e sono tutti accessibili tramite ``symfony console``. Poiché il numero di comandi disponibili potrebbe essere elevato, dovremmo raggrupparli per nome. Per convenzione, i comandi dell'applicazione vengono raggruppati sotto il namespace ``app``. Si possono aggiungere ulteriori namespace, separati con i due punti (``:``).

Un comando riceve un *input* (parametri e opzioni passati al comando) ed è possibile utilizzare il suo *output* per scrivere nella console.

Pulire il database eseguendo il comando:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Impostare un cron su Platform.sh
--------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

Una delle cose belle di Platform.sh è che la maggior parte della configurazione è memorizzata in un unico file: ``.platform.app.yaml``. Il container web, i worker e i cron sono descritti insieme, per facilitare la manutenzione:

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

La sezione ``crons`` definisce tutti i processi di cron. Ogni cron funziona secondo quanto indicato in ``spec``.

Il programma ``croncape`` controlla l'esecuzione del comando e, se questo restituisce un codice di uscita diverso da ``0``, invia un'email agli indirizzi definiti nella variabile d'ambiente ``MAILTO``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Configurare la variabile d'ambiente ``MAILTO``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Si noti che i cron sono impostati su tutti i branch di Platform.sh. Se si desidera escluderne qualcuno in ambienti non di produzione, controllare la variabile d'ambiente ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Andare oltre

    * `Sintassi di cron e crontab`_;

    * `Repository di Croncape`_;

    * `Comandi della console di Symfony`_;

    * `Cheat sheet della console di Symfony`_.

.. _`Sintassi di cron e crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Repository di Croncape`: https://github.com/symfonycorp/croncape
.. _`Comandi della console di Symfony`: https://symfony.com/doc/current/console.html
.. _`Cheat sheet della console di Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
