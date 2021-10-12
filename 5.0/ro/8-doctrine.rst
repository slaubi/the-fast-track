Descrierea structurii datelor
=============================

.. index::
    single: Doctrine
    single: Database

Pentru a gestiona baza de date din PHP, vom depinde de `Doctrine`_, un set de biblioteci care ajută dezvoltatorii să gestioneze bazele de date:

.. code-block:: bash

    $ symfony composer req "orm:^2"

Această comandă instalează câteva dependențe: Doctrine DBAL (o librărie care abstractizează modul de a te conecta la o bază de date), Doctrine ORM (o librărie pentru a manipula conținutul bazei noastre de date folosind obiecte PHP) și Doctrine Migrations.

Configurarea Doctrine ORM
-------------------------

.. index::
    single: Doctrine;Configuration

De unde cunoaște Doctrine datele de acces la baza de date? Rețeta Doctrine a adăugat un fișier de configurare, ``config/packages/doctrine.yaml``, care îi controlează comportamentul. Setarea principală este *DSN-ul bazei de date*, un șir care conține toate informațiile despre conexiune: credențiale, gazdă, port, etc. În mod implicit, Doctrine caută o variabilă de mediu ``DATABASE_URL``.

Înțelegerea convențiilor de variabile ale mediului Symfony
-------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Poți defini manual ``DATABASE_URL`` în fișierul ``.env`` sau ``.env.local``. De fapt, datorită rețetei pachetului, vei vedea un exemplu ``DATABASE_URL`` în fișierul tău ``.env``. Dar, deoarece portul local către PostgreSQL expus de Docker se poate schimba, pot apărea probleme. Există o cale mai bună.

În loc de a scrie ``DATABASE_URL`` direct într-un fișier, putem prefixa toate comenzile cu ``symfony``. Aceasta va detecta serviciile administrate de Docker și / sau SymfonyCloud (când tunelul este deschis) și va seta variabila de mediu automat.

Docker Compose și SymfonyCloud funcționează perfect cu Symfony, datorită acestor variabile de mediu.

.. index::
    single: Symfony CLI;var:export

Verificați toate variabilele de mediu expuse executând ``symfony var:export``:

.. code-block:: bash

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

Îți amintești de *numele serviciului* ``database`` utilizat în configurațiile Docker și SymfonyCloud? Numele serviciilor sunt utilizate ca prefixe pentru a defini variabile de mediu precum ``DATABASE_URL``. Dacă serviciile tale sunt numite în conformitate cu convențiile Symfony, nu este necesară altă configurație.

.. note::

    Bazele de date nu sunt singurul serviciu care beneficiază de convențiile Symfony. Același lucru este valabil și pentru Mailer, de exemplu (prin variabila de mediu ``MAILER_DSN``).

Modificarea valorii implicite DATABASE_URL în .env
---------------------------------------------------

Vom modifica în continuare fișierul ``.env`` pentru a configura ``DATABASE_DSN`` să utilizeze PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -26,5 +26,5 @@ APP_SECRET=7567b803de0f51b0d93e66b064cad2bf
     # 
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=5.7"
    -DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=13&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"
     ###< doctrine/doctrine-bundle ###

De ce informațiile trebuie duplicate în două locuri diferite? Deoarece pe unele platforme Cloud, la *build time* (compilare), URL-ul bazei de date s-ar putea să nu fie cunoscut încă, dar Doctrine trebuie să știe ce bază de date va fi folosită pentru a se configura. Deci, valorile pentru adresa serverului, numele de utilizator și parola nu contează cu adevărat.

Crearea claselor entitate
-------------------------

Obiectul ``Conference`` poate avea următoarele proprietăți:

* *city* - orașul în care este organizată conferința;

* *year* - anul conferinței;

* Un câmp *international* pentru a indica dacă conferința este locală sau internațională (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

Pachetul Maker ne poate ajuta să generăm o entitate (o clasă *Entity*) care să reprezinte o conferință:

.. code-block:: bash
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Această comandă este interactivă: te va ghida prin procesul de adăugare a tuturor câmpurilor de care ai nevoie. Folosește răspunsurile următoare (cele mai multe dintre acestea sunt valorile implicite, așa că poți apăsa tasta „Enter” pentru a le utiliza):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Iată rezultatul executării comenzii:

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

Clasa ``Conference`` a fost stocată sub spațiul de nume ``App\Entity\``.

Comanda a generat, de asemenea, o clasă *repository* pentru Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\Id
    single: Annotations;@ORM\\GeneratedValue
    single: Annotations;@ORM\\Column

Codul generat arată așa (doar o mică parte din fișier este reprodusă aici):

.. code-block:: php
    :caption: src/App/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    /**
     * @ORM\Entity(repositoryClass=ConferenceRepository::class)
     */
    class Conference
    {
        /**
         * @ORM\Id()
         * @ORM\GeneratedValue()
         * @ORM\Column(type="integer")
         */
        private $id;

        /**
         * @ORM\Column(type="string", length=255)
         */
        private $city;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): self
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

Reține că entitatea în sine este o clasă PHP simplă, fără elemente din Doctrine. Adnotările sunt utilizate pentru a adăuga metadate utile pentru Doctrine pentru a facilita maparea elementelor clasei cu elementele tabelei din baza de date.

Doctrine a adăugat o proprietate ``id`` pentru a stoca cheia primară a rândului în tabel. Această cheie (``@ORM\Id()``) este generată automat (``@ORM\GeneratedValue()``) printr-o strategie care depinde de motorul bazei de date utilizat.

.. index::
    single: Command;make:entity

Acum, generează o entitate pentru comentariile conferinței:

.. code-block:: bash
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime||no)

    $ symfony console make:entity Comment

Introdu următoarele răspunsuri:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime``, ``no``.

Relațiile între Entități
----------------------------

.. index::
    single: Command;make:entity

Cele două entități, Conference și Comment, ar trebui să fie legate între ele. Aceasta este o relație de tip *one-to-many* - o conferință poate avea unul, mai multe sau nici un comentariu.

Folosește din nou comanda ``make:entity`` pentru a adăuga această relație la entitatea ``Conference``:

.. code-block:: bash
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

    Dacă introduci ``?`` ca răspuns pentru tip, vei primi toate tipurile de date acceptate:

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
    single: Annotations;@ORM\\ManyToOne
    single: Annotations;@ORM\\JoinColumn
    single: Annotations;@ORM\\OneToMany

Aruncă o privire asupra modificărilor făcute în entitate în urma adăugării relației:

.. code-block:: diff
    :class: ignore

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -36,6 +36,12 @@ class Comment
          */
         private $createdAt;

    +    /**
    +     * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
    +     * @ORM\JoinColumn(nullable=false)
    +     */
    +    private $conference;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +94,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): self
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -31,6 +33,16 @@ class Conference
          */
         private $isInternational;

    +    /**
    +     * @ORM\OneToMany(targetEntity=Comment::class, mappedBy="conference", orphanRemoval=true)
    +     */
    +    private $comments;
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
    +     * @return Collection|Comment[]
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): self
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments[] = $comment;
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): self
    +    {
    +        if ($this->comments->contains($comment)) {
    +            $this->comments->removeElement($comment);
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

Tot ceea ce ai nevoie pentru a gestiona relația a fost generat pentru tine. Odată generat, codul devine al tău; nu ezita să-l personalizezi așa cum dorești.

Adăugarea mai multor proprietăți
-----------------------------------

.. index::
    single: Command;make:entity

Tocmai mi-am dat seama că am uitat să adăugăm o proprietate la entitatea Comment: participanții ar putea dori să atașeze o fotografie a conferinței pentru a ilustra feedback-ul lor.

Execută încă o dată ``make:entity`` și adăugă o proprietate/coloană ``photoFilename`` de tip ``string``, dar permite-i să fie ``null``, deoarece încărcarea unei fotografii este opțională:

.. code-block:: bash
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrarea bazei de date
----------------------

.. index:: ! Command;make:migration

Modelul proiectului este acum complet descris de cele două clase generate.

În continuare, trebuie să creăm tabele de baze de date legate de aceste entități PHP.

*Doctrine Migrations* este instrumentul perfect pentru asta. A fost deja instalat ca parte a dependenței ``orm``.

O *migrare* este o clasă care descrie modificările necesare pentru a actualiza o schemă a bazei de date de la starea ei curentă la cea nouă, așa cum este definită de adnotările din entități. Deoarece baza de date este goală deocamdată, migrația va consta în crearea celor două tabele.

Să vedem ce generează Doctrine:

.. code-block:: bash

    $ symfony console make:migration

Observă numele de fișier generat (un nume care arată ca ``migrations/Version20191019083640.php``):

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version20191019083640 extends AbstractMigration
    {
        public function up(Schema $schema) : void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema) : void
        {
            // ...
        }
    }

Actualizarea bazei de date locale
---------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Acum poți rula migrația generată pentru a actualiza schema bazei de date locale:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Schema bazei de date locale este actualizată, gata să stocheze date.

Actualizarea bazei de date de producție
----------------------------------------

Pașii necesari pentru migrarea bazei de date de producție sunt similari celor cunoscuți deja: salvează modificările și lansează.

La implementarea proiectului, SymfonyCloud actualizează codul, dar execută și migrația bazei de date, dacă există (detectează dacă există comanda ``doctrine:migrations:migrate``).

.. sidebar:: Mergând mai departe

    * `Baze de date și Doctrine ORM <https://symfony.com/doc/current/doctrine.html>`_ în aplicațiile Symfony;

    * `Tutorialul SymphonyCasts Doctrine <https://symfonycasts.com/screencast/symfony-doctrine/install>`_;

    * `Lucrul cu asocierile, relațiile Doctrine <https://symfony.com/doc/current/doctrine/associations.html>`_;

    * `DoctrineMigrationsBundle docs <https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html>`_.

.. _`Doctrine`: https://www.doctrine-project.org/
