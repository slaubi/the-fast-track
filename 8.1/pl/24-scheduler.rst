Planowanie zadań
================

.. index::
    single: Cron

Niektóre zadania konserwacyjne muszą działać według harmonogramu. W przeciwieństwie do robotników (ang. workers), którzy działają nieprzerwanie, zadania zaplanowane uruchamiają się okresowo na krótki czas.

Czyszczenie komentarzy
----------------------

Komentarze oznaczone jako spam lub odrzucone w panelu administracyjnym są przechowywane w bazie danych, ponieważ w przyszłości może zajść potrzeba ich ponownej weryfikacji. Po pewnym czasie powinny raczej zostać usunięte. Wystarczające wydaje się przechowywanie ich przez tydzień od momentu utworzenia.

W repozytorium komentarzy utwórz kilka przydatnych metod, które pozwolą nam na pobranie listy odrzuconych komentarzy, policzenie ich oraz ich całkowite usunięcie:

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

Wszystkie polecenia naszej aplikacji są zarejestrowane wraz z tymi wbudowanymi w Symfony i są dostępne poprzez ``symfony console``. Ponieważ liczba dostępnych poleceń może być duża, grupuj je w przestrzeniach nazw. Zgodnie z konwencją, polecenia aplikacji powinny być przechowywane w przestrzeni nazw ``app``. Możesz również tworzyć dowolną liczbę "podprzestrzeni" poprzez rozdzielenie ich dwukropkiem (``:``).

Komenda deklaruje swoje *argumenty* i *opcje* za pomocą atrybutów ``#[Argument]`` i ``#[Option]`` na parametrach metody ``__invoke()`` (parametr ``$dryRun`` staje się opcją ``--dry-run``). Symfony wstrzykuje pozostałe parametry na podstawie ich typu: ``SymfonyStyle`` do zapisywania ładnie sformatowanych danych w konsoli oraz dowolną usługę, taką jak repozytorium komentarzy, w taki sam sposób, jak robi to dla argumentów kontrolera.

Wyczyść bazę danych uruchamiając polecenie:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Planowanie komendy
------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Uruchamianie polecenia ręcznie działa, ale powinno ono działać każdej nocy. Komponent Symfony Scheduler generuje wiadomości według harmonogramu; są one następnie konsumowane przez robotnika, tak jak każde inne wiadomości Messengera.

Dodaj komponent Scheduler wraz z biblioteką, która parsuje wyrażenia cron:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Zaplanuj polecenie za pomocą atrybutu ``#[AsCronTask]``:

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

Atrybut rejestruje polecenie w domyślnym *harmonogramie* z wyrażeniem cron: każdej nocy o 23:50 (UTC). Sprawdź to:

.. code-block:: terminal

    $ symfony console debug:scheduler

Harmonogram jest udostępniany jako zwykły transport Messengera o tej samej nazwie; konsumuj go jak każdy inny transport:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Wdrażanie harmonogramu
----------------------

.. index::
    single: Upsun;Workers

Na Upsun robotnik konsumuje tylko transport ``async``. Spraw, aby konsumował również harmonogram:

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

To wszystko, czego potrzeba: żadnego crontaba, żadnego dodatkowego procesu; harmonogram żyje w kodzie PHP, obok zadania, które wyzwala, i jest wdrażany oraz wersjonowany jak reszta aplikacji.

Co z systemowymi cronami?
-------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun obsługuje również zadania cron na poziomie systemu operacyjnego, opisane w ``.upsun/config.yaml`` obok kontenera webowego i robotników; domyślna konfiguracja definiuje już jedno, które usuwa wygasłe sesje PHP. Systemowe crony dobrze sprawdzają się w przypadku zadań, które nie są zaimplementowane w PHP.

Narzędzie ``croncape`` używane przez domyślny cron monitoruje wykonanie polecenia i wysyła wiadomość e-mail na adresy zdefiniowane w zmiennej środowiskowej ``MAILTO``, jeśli polecenie zwróci kod inny niż ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Zauważ, że harmonogramy zadań są ustawione na wszystkich gałęziach Upsun. Jeśli nie chcesz uruchamiać niektórych z nich na środowiskach nieprodukcyjnych, sprawdź zmienną środowiskową ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Idąc dalej

    * `Dokumentacja komponentu Scheduler`_;

    * `Składnia cron/crontab`_;

    * `Repozytorium Croncape'a`_;

    * `Polecenia Symfony Console`_;

    * `Ściągawka poleceń konsolowych Symfony`_.

.. _`Dokumentacja komponentu Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`Składnia cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Repozytorium Croncape'a`: https://github.com/symfonycorp/croncape
.. _`Polecenia Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Ściągawka poleceń konsolowych Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
