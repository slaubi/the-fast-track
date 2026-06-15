Opis struktury danych
=====================

.. index::
    single: Doctrine
    single: Database

Do komunikacji PHP z bazą danych użyjemy `Doctrine`_. Jest to zestaw bibliotek, które pomagają zarządzać bazami danych: Doctrine DBAL (warstwa abstrakcji bazy danych), Doctrine ORM (biblioteka do zarządzania danymi w bazie danych, za pomocą obiektów PHP), oraz Doctrine Migrations.

Konfigurowanie Doctrine ORM
---------------------------

.. index::
    single: Doctrine;Configuration

Skąd Doctrine wie, jak połączyć się z bazą danych? Przepis (ang. recipe) instalujący Doctrine dodał odpowiedni plik konfiguracyjny (``config/packages/doctrine.yaml``), który kontroluje jego zachowanie. Głównym ustawieniem jest *database DSN*, napis zawierający wszystkie informacje o połączeniu: dane uwierzytelniające, host, port itp. Domyślnie Doctrine szuka zmiennej środowiskowej ``DATABASE_URL``.

Prawie wszystkie zainstalowane pakiety posiadają swoją konfigurację w katalogu ``config/packages/``. W większości przypadków, wartości domyślne były wybierane ostrożnie, aby działały w większości aplikacji.

Zrozumienie konwencji zmiennych środowiskowych w Symfony
---------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Możesz zdefiniować zmienną środowiskową ``DATABASE_URL`` ręcznie w pliku ``.env`` lub ``.env.local``. Dzięki przepisowi (ang. recipe) pakietu, możesz bazować na przykładowej wartości zmiennej środowiskowej ``DATABASE_URL`` już wpisanej w pliku ``.env``. Pojawia się jednak problem uciążliwej ręcznej aktualizacji wpisu po każdej zmianie portu bazy danych PostgreSQL udostępnionego przez Dockera.  Lepiej więc podejść do sprawy w inny sposób.

Zamiast dokonywać sztywnego ustawienia zmiennej środowiskowej ``DATABASE_URL`` w pliku, możesz poprzedzać wszystkie polecenia słowem ``symfony``. Dzięki temu wszystkie usługi działające w kontenerze Docker i/lub Platform.sh (wyłącznie jeśli mamy otwarty tunel z Platform.sh) będą automatycznie ustawione jako zmienne środowiskowe.

Dzięki zmiennym środowiskowym integracja Symfony z Docker Compose i Platform.sh jest bezproblemowa.

.. index::
    single: Symfony CLI;var:export

Możesz sprawdzić aktualne zmienne środowiskowe w konsoli poprzez użycie polecenia ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Pamiętasz *nazwę usługi* ``database``, której użyliśmy w konfiguracji Docker i Platform.sh? Nazwy usług są używane jako prefiksy do definiowania zmiennych środowiskowych, takich jak ``DATABASE_URL``. Jeśli twoje usługi są nazwane zgodnie z konwencjami Symfony, żadna dodatkowa konfiguracja nie jest potrzebna.

.. note::

    Bazy danych nie są jedyną usługą, która korzysta tej z konwencji. To samo dotyczy na przykład Mailera (zmienna środowiskowa ``MAILER_DSN``).

Zmiana domyślnej wartości DATABASE_URL w pliku .env
-----------------------------------------------------

Zmienimy plik ``.env`` tak, aby ustawić domyślną wartość zmiennej środowiskowej ``DATABASE_URL`` dla PostgreSQL:

.. code-block:: diff

    --- i/.env
    +++ w/.env
    @@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data_%kernel.environment%.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Dlaczego musimy powielać tę samą informację w dwóch różnych miejscach? Ponieważ na niektórych platformach chmurowych, *w czasie budowania* aplikacji, adres URL bazy danych może nie być jeszcze znany, a Doctrine potrzebuje informacji o silniku bazy danych, aby zbudować odpowiednią dla niego konfigurację. Tak więc host, nazwa użytkownika i hasło nie mają większego znaczenia.

Tworzenie klas encji
--------------------

Konferencję można opisać kilkoma atrybutami:

* *city* - miasto, w którym organizowana jest konferencja;

* *year* - rok, w którym odbywa się konferencja;

* *isInternational* - flaga wskazująca, czy konferencja jest krajowa, czy międzynarodowa (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

Maker Bundle pomoże nam wygenerować klasę *encji* (ang. entity), która będzie reprezentowała konferencję.

Pora wygenerować encję ``Conference``:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

To polecenie jest uruchamiane w trybie interaktywnym - poprowadzi Cię przez proces dodawania wszystkich potrzebnych pól. Użyj następujących odpowiedzi (większość z nich to odpowiedzi domyślne, więc możesz nacisnąć klawisz "Enter", aby ich użyć):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Oto pełne wyjście (ang. output) po uruchomieniu polecenia:

.. code-block:: text
    :emphasize-lines: 8,11,14,17,22,25,28,31,36,39,42,45
    :class: ignore

     created: src/Entity/Conference.php
     created: src/Repository/ConferenceRepository.php

     Entity generated! Now let's add some fields!
     You can always add more fields later manually or by re-running this command.

     New property name (press <return> to stop adding fields):
     > city

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > year

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     > 4

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > isInternational

     Field type (enter ? to see all types) [boolean]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     >



      Success!


     Next: When you're ready, create a migration with make:migration

Klasa ``Conference`` została zapisana w przestrzeni nazw ``App\Entity\``.

Polecenie wygenerowało również klasę *repozytorium* (ang. repository) Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Wygenerowany kod wygląda następująco (tylko niewielka część pliku jest tu pokazana):

.. code-block:: php
    :caption: src/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    class Conference
    {
        #[ORM\Id]
        #[ORM\GeneratedValue]
        #[ORM\Column]
        private ?int $id = null;

        #[ORM\Column(length: 255)]
        private ?string $city = null;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): static
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

Zauważ, że właśnie utworzona klasa jest zwykłą klasą PHP - nie ma w niej elementów Doctrine. Metadane wykorzystywane przez Doctrine do powiązania klasy z tabelą w bazie danych dodajemy, używając atrybutów.

Doctrine dodał atrybut ``id``, aby zachować klucz główny w tabeli bazy danych. Ten klucz (``ORM\Id()``) jest automatycznie generowany (``ORM\GeneratedValue()``) w sposób zależny od silnika bazy danych (oparty o wzorzec strategii).

.. index::
    single: Command;make:entity

Teraz wygeneruj klasę encji dla komentarzy:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Wprowadź następujące odpowiedzi:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

Łączenie encji
----------------

.. index::
    single: Command;make:entity

Obie encje, ``Conference`` i ``Comment``, powinny być ze sobą powiązane. Konferencja może mieć zero lub więcej komentarzy, co nazywamy relacją *jeden do wielu* (ang. one to many).

Użyj ponownie polecenia ``make:entity``, aby zdefiniować tę relację w klasie ``Conference``:

.. code-block:: terminal
    :class: answers(comments||OneToMany||Comment||conference||no||yes)

    $ symfony console make:entity Conference

.. code-block:: text
    :emphasize-lines: 4,7,10,15,18,27
    :class: ignore

     Your entity already exists! So let's add some new fields!

     New property name (press <return> to stop adding fields):
     > comments

     Field type (enter ? to see all types) [string]:
     > OneToMany

     What class should this entity be related to?:
     > Comment

     A new property will also be added to the Comment class...

     New field name inside Comment [conference]:
     >

     Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
     > no

     Do you want to activate orphanRemoval on your relationship?
     A Comment is "orphaned" when it is removed from its related Conference.
     e.g. $conference->removeComment($comment)

     NOTE: If a Comment may *change* from one Conference to another, answer "no".

     Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
     > yes

     updated: src/Entity/Conference.php
     updated: src/Entity/Comment.php

.. note::

    Jeśli wpiszesz ``?`` jako odpowiedź w pytaniu o typ pola, otrzymasz listę wszystkich obsługiwanych typów:

    .. code-block:: text
        :class: ignore

        Main types
          * string
          * text
          * boolean
          * integer (or smallint, bigint)
          * float

        Relationships / Associations
          * relation (a wizard will help you build the relation)
          * ManyToOne
          * OneToMany
          * ManyToMany
          * OneToOne

        Array/Object Types
          * array (or simple_array)
          * json
          * object
          * binary
          * blob

        Date/Time Types
          * datetime (or datetime_immutable)
          * datetimetz (or datetimetz_immutable)
          * date (or date_immutable)
          * time (or time_immutable)
          * dateinterval

        Other Types
          * decimal
          * guid
          * json_array

.. index::
    single: Attributes;ORM\\ManyToOne
    single: Attributes;ORM\\JoinColumn
    single: Attributes;ORM\\OneToMany

Przyjrzyj się liście różnic dla klas encji po dodaniu relacji:

.. code-block:: diff
    :class: ignore

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -23,6 +23,10 @@ class Comment
         #[ORM\Column]
         private ?\DateTimeImmutable $createdAt = null;

    +    #[ORM\ManyToOne(inversedBy: 'comments')]
    +    #[ORM\JoinColumn(nullable: false)]
    +    private ?Conference $conference = null;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +92,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): static
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -20,6 +22,19 @@ class Conference
         #[ORM\Column]
         private ?bool $isInternational = null;

    +    /**
    +     * @var Collection<int, Comment>
    +     */
    +    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
    +    private Collection $comments;
    +
    +    public function __construct()
    +    {
    +        $this->comments = new ArrayCollection();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -71,4 +83,35 @@ class Conference

             return $this;
         }
    +
    +    /**
    +     * @return Collection<int, Comment>
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): static
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments->add($comment);
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): static
    +    {
    +        if ($this->comments->removeElement($comment)) {
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

Wszystko, czego potrzebujesz do zarządzania tą relacją, zostało wygenerowane automatycznie. Później kod możesz zmieniać jak chcesz.

Dodawanie kolejnych atrybutów
------------------------------

.. index::
    single: Command;make:entity

Właśnie zdałem sobie sprawę, że zapomnieliśmy dodać pewien atrybut w encji ``Comment``: uczestnicy mogą chcieć dołączyć zdjęcie z konferencji, aby zilustrować swoje opinie.

Uruchom ``make:entity`` jeszcze raz i dodaj atrybut ``photoFilename`` jako kolumnę typu ``string``. Pozwól jej przyjmować wartość ``null``, ponieważ dodanie zdjęcia jest opcjonalne:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migracja bazy danych
--------------------

.. index:: ! Command;make:migration

Model projektu składa się teraz z dwóch właśnie wygenerowanych klas.

W kolejnym kroku musimy utworzyć tabele w bazie danych związane z naszymi encjami w PHP.

Biblioteka *Doctrine Migrations* to narzędzie idealnie dopasowane do tego zadania. Została ona już zainstalowana jako część zależności ``orm``.

*Migracja* (ang. migration) jest klasą, która opisuje zmiany wykonywane w bazie danych, aby z obecnego schematu przejść na nowy, zdefiniowany w atrybutach encji. Ponieważ baza danych jest na razie pusta, migracja powinna składać się z operacji tworzących dwie tabele.

Zobaczmy, co wygeneruje Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Zwróć uwagę na wygenerowaną nazwę pliku, która powinna przypominać ``migrations/Version20191019083640.php``:

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version00000000000000 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema): void
        {
            // ...
        }
    }

Aktualizacja lokalnej bazy danych
---------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Możesz teraz uruchomić migrację, aby zaktualizować schemat lokalnej bazy danych:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Schemat lokalnej bazy danych jest teraz aktualny i przygotowany do przechowywania niektórych danych.

Aktualizacja produkcyjnej bazy danych
-------------------------------------

Kroki potrzebne do wykonania migracji na produkcyjnej bazie danych są takie same jak te, które już znasz: zatwierdź zmiany (ang. commit) i wdrażaj.

Podczas wdrażania projektu, Platform.sh oprócz aktualizacji kodu uruchamia także migrację bazy danych, jeśli taka istnieje (wykrywa, czy istnieje polecenie ``doctrine:migrations:migrate``).

.. sidebar:: Idąc dalej

    * `Bazy danych i Doctrine ORM`_ w aplikacjach Symfony;

    * `Samouczek SymfonyCasts Doctrine`_;

    * `Wykorzystanie asocjacji i relacji Doctrine`_;

    * `Dokumentacja DoctrineMigrationsBundle`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Bazy danych i Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`Samouczek SymfonyCasts Doctrine`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Wykorzystanie asocjacji i relacji Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`Dokumentacja DoctrineMigrationsBundle`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
