Rulând sarcinile Cron
======================

.. index::
    single: Cron

Sarcinile Cron sunt utile pentru a executa sarcini de întreținere. Spre deosebire de procesele executate constant, acestea au program de executare pentru o perioadă scurtă de timp.

Curățarea comentariilor
-------------------------

Comentariile marcate ca spam sau respinse de administrator sunt păstrate în baza de date, deoarece administratorul ar putea dori să le inspecteze pentru un timp. Dar probabil că ar trebui eliminate după ceva timp. Păstrarea lor în jur la o săptămână după crearea lor este probabil suficient.

Creează câteva metode de utilitate în depozitul de comentarii pentru a găsi comentarii respinse, numără-le și șterge-le:

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
    @@ -16,6 +17,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -23,6 +26,29 @@ class CommentRepository extends ServiceEntityRepository
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
    +                'date' => new \DateTime(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    Pentru interogări mai complexe, uneori este util să aruncăm o privire asupra instrucțiunilor SQL generate (acestea pot fi găsite în jurnale și în depanatorul solicitărilor Web).

Folosind constante de clasă, parametri de container și variabile de mediu
---------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 zile? Am fi putut alege un alt număr, poate 10 sau 20. Acest număr ar putea evolua în timp. Am decis să îl stocăm ca o constantă pe clasă, dar am fi putut să o stocăm ca parametru în container, sau am fi putut chiar să o definim ca o variabilă de mediu.

Iată câteva reguli pentru a decide ce abstractizare trebuie să folosești:

* Dacă valoarea este secretă (parole, token API, ...), utilizează *stocarea secretizată* Symfony sau un seif;

* Dacă valoarea este dinamică și ar trebui să poți să o schimbi *fără* relansare, folosește o variabilă *de mediu*;

* Dacă valoarea poate fi diferită între medii, utilizează un *parametru container*;

* Pentru orice altceva, păstrează valoarea în cod, de exemplu într-o constantă *de clasă*.

Crearea unei comenzi CLI
------------------------

Eliminarea comentariilor vechi este sarcina perfectă pentru o lucrare cron. Ar trebui să se facă în mod regulat, iar o mică întârziere nu are niciun impact major.

Creează o comandă CLI numită ``app:comment:cleanup`` prin crearea unui fișier ``src/Command/CommentCleanupCommand.php``:

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    class CommentCleanupCommand extends Command
    {
        private $commentRepository;

        protected static $defaultName = 'app:comment:cleanup';

        public function __construct(CommentRepository $commentRepository)
        {
            $this->commentRepository = $commentRepository;

            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->setDescription('Deletes rejected and spam comments from the database')
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

            return 0;
        }
    }

Toate comenzile aplicației sunt înregistrate alături de cele incorporate în Symfony și sunt accesibile prin ``consola symfony``. Deoarece numărul de comenzi disponibile poate fi mare, ar trebui să le categorizezi. Prin convenție, comenzile aplicației ar trebui stocate sub spațiul de nume ``app``. Adaugă orice număr de categorii prin separarea lor cu două puncte (``:``).

O comandă primește *intrarea* (argumentele și opțiunile transmise comenzii) și poți utiliza *ieșirea* pentru a scrie în consolă.

Curăță baza de date rulând comanda:

.. code-block:: bash

    $ symfony console app:comment:cleanup

Configurarea unei sarcini Cron pe SymfonyCloud
----------------------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

Unul dintre lucrurile frumoase al SymfonyCloud este că cea mai mare parte a configurației este stocată într-un singur fișier: ``.symfony.cloud.yaml``. Containerul web, procesele și sarcinile cron sunt descrise împreună pentru a ajuta la întreținere:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -52,6 +52,15 @@ hooks:

             (>&2 symfony-deploy)

    +crons:
    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$SYMFONY_BRANCH" = "master" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messages:
             commands:

Secțiunea ``crons`` definește toate sarcinile cron. Fiecare cron rulează conform unui grafic ``spec``.

Utilitarul ``croncape`` monitorizează execuția comenzii și trimite un e-mail la adresele definite în variabila de mediu ``MAILTO`` dacă comanda returnează un cod de ieșire diferit de ``0``.

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

Configurează variabila de mediu ``MAILTO``:

.. code-block:: bash

    $ symfony var:set MAILTO=ops@example.com

Poți forța un cron să ruleze de pe mașina locală:

.. code-block:: bash
    :class: ignore

    $ symfony cron comment_cleanup

Notă: sarcinile cron sunt setate pe toate ramurile SymfonyCloud. Dacă nu dorești să rulezi unele pe medii non-producție, verifică variabila de mediu ``$SYMFONY_BRANCH``:

.. code-block:: bash
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Mergând mai departe

    * `Sintaxa Cron/crontab <https://en.wikipedia.org/wiki/Cron>`_;

    * `Repozitoriu Croncape <https://github.com/symfonycorp/croncape>`_;

    * `Comenzi Symfony Console <https://symfony.com/doc/current/console.html>`_;

    * Referințe `Symfony Console` <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_.
