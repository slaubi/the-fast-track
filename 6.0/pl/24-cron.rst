Harmonogram zadań (ang. cron)
==============================

.. index::
    single: Cron

Harmonogram zadań jest bardzo przydatny do wykonywania cyklicznych operacji, np. zadań konserwacyjnych, ponieważ uruchamiane są one według harmonogramu i pracują przez krótki okres czasu, inaczej niż w przypadku wywoływania robotników (ang. workers).

Czyszczenie komentarzy
----------------------

Komentarze oznaczone jako spam lub odrzucone w panelu administracyjnym są przechowywane w bazie danych, ponieważ w przyszłości może zajść potrzeba ich ponownej weryfikacji. Po pewnym czasie powinny raczej zostać usunięte. Wystarczające wydaje się przechowywanie ich przez tydzień od momentu utworzenia.

W repozytorium komentarzy utwórz kilka przydatnych metod, które pozwolą nam na pobranie listy odrzuconych komentarzy, policzenie ich oraz ich całkowite usunięcie:

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
    +                'date' => new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    W przypadku bardziej złożonych zapytań, czasami przydatne jest zapoznanie się z wygenerowanymi instrukcjami SQL (można je znaleźć w logach oraz w profilerze, w zakładce Doctrine).

Korzystanie ze stałych zawartych w klasach, parametrów kontenera i zmiennych środowiskowych.
-----------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

Siedem dni? Mogliśmy wybrać inną liczbę, może 10 albo 20. Ta liczba może zmieniać się z upływem czasu. Zdecydowaliśmy się przechowywać ją jako stałą w klasie, ale moglibyśmy przechowywać ją również jako parametr w kontenerze a nawet zdefiniować jako zmienną środowiskową.

Oto kilka zasad decydujących o tym, jakiej abstrakcji użyć:

* Jeśli wartość zaliczana jest do tzw. danych wrażliwych, takich jak hasła czy klucze API, użyj magazynu danych poufnych (ang. secret storage) Symfony lub sejfu (ang. vault);

* Jeśli wartość jest dynamiczna i chcesz ją zmienić *bez* ponownego wdrażania aplikacji, użyj *zmiennej środowiskowej*;

* Jeśli wartość może być różna w różnych środowiskach, należy użyć *parametru kontenera*;

* W pozostałych przypadkach przechowuj tę wartość w kodzie, np. jako *stałą w klasie*.

Tworzenie komendy wiersza poleceń
----------------------------------

Usuwanie starych komentarzy to działanie idealne do wykorzystania harmonogramu zadań. Powinno się to odbywać regularnie, a niewielkie opóźnienie nie ma większego znaczenia.

Utwórz komendę linii poleceń o nazwie ``app:comment:cleanup`` przez utworzenie pliku ``src/Command/CommentCleanupCommand.php``:

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

Wszystkie polecenia naszej aplikacji są zarejestrowane wraz z tymi wbudowanymi w Symfony i są dostępne poprzez ``symfony console``. Ponieważ liczba dostępnych poleceń może być duża, grupuj je w przestrzeniach nazw. Zgodnie z konwencją, polecenia aplikacji powinny być przechowywane w przestrzeni nazw ``app``. Możesz również tworzyć dowolną liczbę "podprzestrzeni" poprzez rozdzielenie ich dwukropkiem (``:``).

Komenda linii poleceń dostaje *wejście* (argumenty i opcje przekazywane do komendy) oraz możesz użyć *wyjścia* w celu wyświetlenia danych w konsoli.

Wyczyść bazę danych uruchamiając polecenie:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Konfigurowanie harmonogramu zadań w usłudze Platform.sh
---------------------------------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

Jedną z zalet Platform.sh jest to, że większość konfiguracji jest przechowywana w jednym pliku: ``.platform.app.yaml``. Kontenery serwisów, robotnicy czy zadania z harmonogramu są opisane razem, aby ułatwić zarządzanie nimi:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -60,6 +60,14 @@ crons:
             spec: '50 23 * * *'
             cmd: if [ "$PLATFORM_BRANCH" = "main" ]; then croncape php-security-checker; fi

    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$PLATFORM_BRANCH" = "master" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messenger:
             commands:

Sekcja ``crons`` definiuje wszystkie zadania z harmonogramu. Każde zadanie działa zgodnie z planem ``spec``.

Narzędzie ``croncape`` monitoruje wykonanie polecenia i wysyła wiadomość e-mail na adresy zdefiniowane w zmiennej środowiskowej ``MAILTO``, jeśli polecenie zwróci kod inny niż ``0``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Skonfiguruj zmienną środowiskową ``MAILTO``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Zauważ, że harmonogramy zadań są ustawione na wszystkich gałęziach Platform.sh. Jeśli nie chcesz uruchamiać niektórych z nich na środowiskach nieprodukcyjnych, sprawdź zmienną środowiskową ``$PLATFORM_BRANCH``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Idąc dalej

    * `Składnia cron/crontab`_;

    * `Repozytorium Croncape'a`_;

    * `Polecenia Symfony Console`_;

    * `Ściągawka poleceń konsolowych Symfony`_.

.. _`Składnia cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Repozytorium Croncape'a`: https://github.com/symfonycorp/croncape
.. _`Polecenia Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Ściągawka poleceń konsolowych Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
