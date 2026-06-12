Die Datenstruktur beschreiben
=============================

.. index::
    single: Doctrine
    single: Database

Um in PHP mit der Datenbank umzugehen, werden wir `Doctrine`_ verwenden, eine Reihe von Bibliotheken, die Entwickler*innen helfen, Datenbanken zu verwalten: Doctrine DBAL (eine Datenbank-Abstraktions-Schicht), Doctrine ORM (eine Bibliothek um unseren Datenbank-Inhalt anzupassen mit Hilfe von PHP-Objekten) und Doctrine Migrations.

Doctrine ORM konfigurieren
--------------------------

.. index::
    single: Doctrine;Configuration

Woher kennt Doctrine die Datenbankverbindung? Das Doctrine-Recipe hat eine Konfigurationsdatei hinzugefügt, ``config/packages/doctrine.yaml``, die das Verhalten steuert. Die wichtigste Einstellung ist die *Datenbank-DSN*, eine Zeichenkette, die alle Informationen über die Verbindung enthält: Anmeldeinformationen, Host, Port, etc. Standardmäßig sucht Doctrine nach der Environment-Variable ``DATABASE_URL``.

Fast alle installierten Pakete haben eine Konfigurationsdatei im ``config/packages/``-Verzeichnis. Normalerweise sind die Standardeinstellungen so gewählt, dass sie für die meisten Anwendungen funktionieren.

Konventionen für Symfony-Environment-Variablen verstehen
---------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Du kannst ``DATABASE_URL`` manuell in der ``.env``- oder ``.env.local``-Datei definieren. Dank des Recipes des Paketes siehst Du sogar eine beispielhafte ``DATABASE_URL`` in Deiner ``.env``-Datei. Aber da sich der lokale Port auf PostgreSQL, der von Docker festgelegt wird, ändern kann, ist dieser Weg recht umständlich. Es gibt einen besseren Weg.

Anstatt in einer Datei fest ``DATABASE_URL`` einzusetzen, können wir alle Befehle mit ``symfony`` prefixen. Dadurch werden Dienste erkannt, die von Docker und/oder Upsun ausgeführt werden (wenn der Tunnel geöffnet ist) und die Environment-Variable wird automatisch gesetzt.

Docker Compose und Upsun arbeiten dank dieser Environment-Variablen nahtlos mit Symfony zusammen.

.. index::
    single: Symfony CLI;var:export

Du überprüfst alle exponierten Environment-Variablen, indem Du ``symfony var:export`` ausführst:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Erinnerst Du dich an den ``database``-*Servicenamen*, der in den Konfigurationen von Docker und Upsun verwendet wird? Die Servicenamen werden als Präfixe für Environment-Variablen wie ``DATABASE_URL`` verwendet. Wenn Deine Services nach den Symfony-Konventionen benannt sind, ist keine weitere Konfiguration erforderlich.

.. note::

    Datenbanken sind nicht der einzige Service, der von den Symfony-Konventionen profitiert. Das Gleiche gilt z. B. für Mailer (über die Environment-Variable ``MAILER_DSN``).

Den Standardwert DATABASE_URL in .env ändern
---------------------------------------------

Wir werden die ``.env``-Datei dennoch ändern, um die Standard-``DATABASE_URL`` für die Verwendung von PostgreSQL festzulegen:

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

Warum müssen die Informationen an zwei verschiedenen Stellen dupliziert werden? Da auf einigen Cloud-Plattformen zum *Zeitpunkt des Builds* die Datenbank-URL möglicherweise noch nicht bekannt ist, muss Doctrine die Engine der Datenbank kennen, um ihre Konfiguration zu erstellen. Daher sind der Host und die Zugangsdaten nicht wirklich wichtig.

Entity-Klassen anlegen
----------------------

Eine Konferenz kann mit einigen wenigen Eigenschaften beschrieben werden:

* Die *Stadt*, in der die Konferenz organisiert wird;

* Das *Jahr* der Konferenz;

* Ein *international*-Flag, die angibt, ob die Konferenz lokal oder international ist (SymfonyLive vs. SymfonyCon).

.. index:: ! Command;make:entity

Das Maker-Bundle kann uns helfen, eine Klasse (eine *Entity-Klasse*) zu generieren, die eine Konferenz repräsentiert.

Jetzt ist es an der Zeit die ``Conference``-Entity zu generieren:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Dieser Befehl ist interaktiv: Er führt Dich durch den Prozess des Hinzufügens aller benötigten Felder. Verwende die folgenden Antworten (die meisten davon sind die Standardwerte, Du kannst die Taste "Enter" drücken, um sie zu verwenden):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Das ist die vollständige Ausgabe des Befehls:

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

Die ``Conference``-Klasse wurde unter dem ``App\Entity\``-Namespace abgelegt.

Der Befehl erzeugte auch eine Doctrine *Repository-Klasse*: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Der generierte Code sieht wie folgt aus (nur ein kleiner Teil der Datei wird hier gezeigt):

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

Beachte, dass die Klasse selbst eine einfache PHP-Klasse ohne Anzeichen von Doctrine ist. Mittels Attributen werden Metadaten hinzugefügt, die Doctrine verwendet, um die Klasse der zugehörigen Datenbanktabelle zuzuordnen.

Doctrine hat ein ``id``-Property/Spalte hinzugefügt, um den Primärschlüssel der Zeile in der Datenbanktabelle zu speichern. Dieser Schlüssel (``ORM\Id()``) wird abhängig vom verwendeten Datenbanksystem automatisch generiert (``ORM\GeneratedValue()``).

.. index::
    single: Command;make:entity

Erzeuge nun eine Entity-Klasse für Konferenzkommentare:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Gebe die folgenden Antworten ein:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

Entities miteinander verknüpfen
--------------------------------

.. index::
    single: Command;make:entity

Die beiden Entities, Conference und Comment, sollten miteinander verbunden werden. Eine Konferenz kann null oder mehr Kommentare haben, was als *One-to-Many-Beziehung* bezeichnet wird.

Verwende erneut den ``make:entity``-Befehl, um diese Beziehung zur ``Conference``-Klasse hinzuzufügen:

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

    Wenn Du als Antwort auf den Typ ``?`` eingibst, erhältst Du alle unterstützten Typen:

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

Wirf einen Blick auf das vollständige Diff für die Entity-Klassen, nachdem Du die Beziehung (Relation) hinzugefügt hast:

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

Alles, was Du für die Verwaltung von relations benötigst, wurde für Dich generiert. Sobald der Code generiert ist, gehört er Dir; zöger nicht, ihn nach Deinen Wünschen anzupassen.

Weitere Properties (Spalten) hinzufügen
----------------------------------------

.. index::
    single: Command;make:entity

Mir ist gerade aufgefallen, dass wir vergessen haben, ein Property zur Comment-Entity hinzuzufügen: Die Teilnehmer*innen möchten vielleicht ein Foto der Konferenz anhängen, um ihr Feedback zu veranschaulichen.

Führe ``make:entity`` noch einmal aus und füge ein ``photoFilename`` Property/Spalte vom Typ ``string`` hinzu, aber lass es ``null`` sein, da das Hochladen eines Fotos optional ist:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Die Datenbank migrieren
-----------------------

.. index:: ! Command;make:migration

Das Projektmodell wird nun durch die beiden generierten Klassen vollständig beschrieben.

Als nächstes müssen wir die Datenbanktabellen erstellen, die sich auf diese PHP-Entitys beziehen.

*Doctrine Migrations* ist die perfekte Ergänzung für eine solche Aufgabe. Es wurde bereits als Teil der ``orm``-Dependency installiert.

Eine *Migration* ist eine Klasse, welche die Änderungen beschreibt, die erforderlich sind, um ein Datenbankschema von seinem aktuellen Zustand auf den neuen Zustand, der durch die Attribute in den Entities definiert ist, zu aktualisieren. Da die Datenbank vorerst leer ist, sollte die Migration aus zwei Einträgen bestehen.

Mal sehen, was Doctrine erzeugt:

.. code-block:: terminal

    $ symfony console make:migration

Beachte den generierten Dateinamen in der Ausgabe (ein Name, der so aussieht ``migrations/Version20191019083640.php``):

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

Die lokale Datenbank aktualisieren
----------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Du kannst nun die generierte Migration ausführen, um das lokale Datenbankschema zu aktualisieren:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Das lokale Datenbankschema ist nun auf dem aktuellen Stand und bereit, einige Daten zu speichern.

Die Datenbank der Produktivumgebung aktualisieren
-------------------------------------------------

Die Schritte, die für die Migration der Datenbank für die Produktivumgebung erforderlich sind, sind die gleichen wie die, mit denen Du bereits vertraut bist: Committe die Änderungen und deploye.

Beim Deployment des Projekts aktualisiert Upsun den Code, führt aber auch die Datenbankmigration durch, falls vorhanden (Upsun erkennt, ob der ``doctrine:migrations:migrate``-Befehl existiert).

.. sidebar:: Weiterführendes

    * `Datenbanken und Doctrine ORM`_ in Symfony-Anwendungen;

    * `SymfonyCasts Doctrine Tutorial`_;

    * `Arbeiten mit Doctrine Associations/Relations`_;

    * `DoctrineMigrationsBundle Dokumentation`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Datenbanken und Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`SymfonyCasts Doctrine Tutorial`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Arbeiten mit Doctrine Associations/Relations`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle Dokumentation`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
