Beschrijving van de gegevensstructuur
=====================================

.. index::
    single: Doctrine
    single: Database

Om in PHP met een database te werken gaan we gebruik maken van `Doctrine`_ , een set libraries die ons, ontwikkelaars, helpt om met databases om te gaan: Doctrine DBAL (een database abstractie laag), Doctrine ORM (een library om database inhoud te manipuleren door gebruik te maken van PHP objecten), en Doctrine Migrations.

Doctrine ORM configureren
-------------------------

.. index::
    single: Doctrine;Configuration

Hoe weet Doctrine met welke database te verbinden? Doctrine's recipe voegde een configuratiebestand toe ``config/packages/doctrine.yaml``, dat zijn gedrag regelt. De belangrijkste instelling is de *database DSN*, een string die alle informatie over de verbinding bevat: gebruikersnaam, wachtwoord, host, poort, enz. Standaard probeert Doctrine deze gegevens uit de ``DATABASE_URL`` omgevingsvariabele te halen.

Bijna alle geïnstalleerde packages bevatten configuratie in de ``config/packages/``-map. Meestal zijn de standaardinstellingen zorgvuldig gekozen om voor de meeste toepassingen te werken.

Conventies van Symfony-omgevingsvariabelen begrijpen
----------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Je kan de ``DATABASE_URL`` handmatig in het ``.env`` of ``.env.local`` bestand definiëren. Dankzij het recipe van de package zie je bijvoorbeeld ``DATABASE_URL`` in jouw ``.env`` bestand. Maar omdat Docker de lokale poort voor PostgreSQL vrij kiest, is dat hinderlijk. Er is een betere manier.

In plaats van de ``DATABASE_URL`` hard te coderen in een bestand, kunnen we alle commando's met ``symfony`` prefixen. Dit zorgt ervoor dat de Docker en/of Platform.sh (wanneer de tunnel open is) services gedetecteerd worden en automatisch als omgevingsvariabele ingesteld worden.

Docker Compose en Platform.sh werken naadloos samen met Symfony dankzij deze omgevingsvariabelen.

.. index::
    single: Symfony CLI;var:export

Bekijk alle beschikbare omgevingsvariabelen door het uitvoeren van ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

Herinner je je de ``database`` *servicenaam* die in de Docker en Platform.sh configuraties wordt gebruikt? De servicenamen worden gebruikt als prefix bij het definiëren van omgevingsvariabelen zoals ``DATABASE_URL``. Als services de Symfony naamgevingsconventies volgen, is er geen extra configuratie nodig.

.. note::

    De database is niet de enige service die profiteert van de Symfony conventies. Hetzelfde geldt bijvoorbeeld voor Mailer (via de ``MAILER_DSN`` omgevingsvariabele).

De standaard DATABASE_URL waarde in .env aanpassen
--------------------------------------------------

We zullen het ``.env`` bestand nog steeds aanpassen om de standaard ``DATABASE_URL`` voor het gebruik van PostgreSQL in te stellen:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -29,7 +29,7 @@ MESSENGER_TRANSPORT_DSN=doctrine://default?auto_setup=0
     #
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=14&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=14&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Waarom moet de informatie op twee verschillende plaatsen worden herhaald? Omdat op sommige Cloud platformen tijdens de *build*, de URL van de database nog onbekend kan zijn, maar Doctrine wel het database systeem moet kennen om de configuratie te kunnen opbouwen. Dus, de host, gebruikersnaam en wachtwoord doen er niet toe.

Entity classes aanmaken
-----------------------

Een conferentie kunnen we beschrijven aan de hand van een aantal eigenschappen:

* De *stad* waar de conferentie wordt georganiseerd;

* Het *jaar* van de conferentie;

* Een *internationale* vlag om aan te geven of de conferentie lokaal of internationaal is (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

De Maker bundle kan ons helpen om een class (een *Entity* class) te genereren voor de conferentie.

Het is nu tijd om de ``Conference`` entity te genereren:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Dit commando is interactief: het begeleidt je bij het toevoegen van de nodige velden. Gebruik de volgende antwoorden (de meeste zijn de standaard antwoorden, dus je kunt op de "Enter" toets drukken om ze te aanvaarden):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Dit is de volledige uitvoer van het commando:

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

De ``Conference`` class is opgeslagen onder de ``App\Entity\`` namespace.

Het commando genereerde ook een Doctrine *repository* class: ``App\Repository\ConferenceRepository`` .

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

De gegenereerde code ziet er als volgt uit (slechts een klein deel van het bestand wordt hier getoond):

.. code-block:: php
    :caption: src/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    class Conference
    {
        #[ORM\Column(type: 'integer')]
        #[ORM\Id, ORM\GeneratedValue()]
        private $id;

        #[ORM\Column(type: 'string', length: 255)]
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

Merk op dat de class een gewone PHP class is zonder invloeden van Doctrine. Attributen worden gebruikt om metadata toe te voegen die Doctrine gebruikt om de class te kunnen koppelen aan de bijhorende databasetabel.

Doctrine heeft een ``id`` eigenschap toegevoegd om de primaire sleutel van de rij te bewaren in de tabel. Deze sleutel (``ORM\Id()``) wordt automatisch gegenereerd (``ORM\GeneratedValue()``) via een strategie die afhankelijk is van het gebruikte databasesysteem.

.. index::
    single: Command;make:entity

Genereer nu een Entity class voor reacties op de conferentie:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Geef de volgende antwoorden:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

Entities aan elkaar koppelen
----------------------------

.. index::
    single: Command;make:entity

De twee entities, Conference en Comment, moeten aan elkaar worden gekoppeld. Een conferentie kan nul of meer reacties hebben, wat een *one-to-many*-relatie wordt genoemd.

Gebruik het ``make:entity`` commando opnieuw om de relatie toe te voegen aan de ``Conference`` class:

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

    Als je ``?`` als antwoord voor het type intypt, krijg je een lijst met alle ondersteunde types:

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

Bekijk de volledige diff van de entity class na het toevoegen van de relatie:

.. code-block:: diff
    :class: ignore

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -36,6 +36,12 @@ class Comment
          */
         private $createdAt;

    +    #[ORM\ManyToOne(inversedBy: 'comments')]
    +    #[ORM\JoinColumn(nullable: false)]
    +    private Conference $conference;
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

    +    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: "conference", orphanRemoval: true)]
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
    +     * @return Collection<int, Comment>
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

Alles wat nodig is om de relatie te beheren is nu voor je gegenereerd. Eenmaal gegenereerd, wordt dit jouw code; je bent vrij om de code aan te passen als dat nodig is.

Extra eigenschappen toevoegen
-----------------------------

.. index::
    single: Command;make:entity

Ik realiseerde me net dat we vergeten zijn een eigenschap toe te voegen aan de Comment entity: de deelnemers willen misschien een foto van de conferentie toevoegen om hun feedback kracht bij te zetten.

Voer ``make:entity`` opnieuw uit en voeg een ``photoFilename`` eigenschap/kolom van het type ``string`` toe, maar laat ``null`` toe omdat het uploaden van een foto optioneel is:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

De database migreren
--------------------

.. index:: ! Command;make:migration

Het model van het project wordt nu volledig beschreven door de twee gegenereerde classes.

Vervolgens moeten we ook nog de databasetabellen aanmaken die bij deze PHP entities horen.

*Doctrine Migrations* is hiervoor het beste gereedschap. Het werd al geïnstalleerd als onderdeel van de ``orm`` dependency.

Een *migratie* is een class die database schemawijzigingen beschrijft. Met die schemawijzigingen kan je de database van de huidige naar de nieuwe versie brengen. De schemawijzigingen worden gegenereerd op basis van de attributen die op de entity gedefinieerd zijn. De database is momenteel leeg, dus de migratie zou de creatie van twee tabellen moeten bevatten.

Laten we eens bekijken wat Doctrine genereert:

.. code-block:: terminal

    $ symfony console make:migration

Let op de gegenereerde bestandsnaam (ziet eruit als ``migrations/Version20191019083640.php`` ):

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

Bijwerken van de lokale database
--------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Je kan nu de gegenereerde migratie uitvoeren om het lokale database schema bij te werken:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Het lokale database-schema is nu up-to-date, klaar om gegevens te bewaren.

De productiedatabase bijwerken
------------------------------

De stappen die nodig zijn om de productiedatabase te migreren zijn dezelfde als die waarmee je al bekend bent: commit de wijzigingen en deploy deze.

Bij het deployen van het project brengt Platform.sh de code up-to-date en voert ook de databasemigraties uit (indien het ``doctrine:migrations:migrate`` commando bestaat).

.. sidebar:: Verder gaan

    * `Databases en Doctrine ORM`_ in Symfony applicaties;

    * `SymfonyCasts Doctrine tutorial`_;

    * `Werken met Doctrine Associations/Relations`_;

    * `DoctrineMigrationsBundle documentatie`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Databases en Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`SymfonyCasts Doctrine tutorial`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Werken met Doctrine Associations/Relations`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle documentatie`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
