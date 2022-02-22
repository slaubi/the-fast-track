Descrivere la struttura dati
============================

.. index::
    single: Doctrine
    single: Database

Per gestire il database da PHP, il progetto dipenderà da `Doctrine`_, un insieme di librerie che aiutano gli sviluppatori a gestire i database: Doctrine DBAL (un layer di astrazione per database), Doctrine ORM (una libreria per manipolare il contenuto del database utilizzando oggetti PHP), e Doctrine Migrations.

Configurare l'ORM di Doctrine
-----------------------------

.. index::
    single: Doctrine;Configuration

Come fa Doctrine a conoscere la connessione al database? La ricetta di Doctrine ha aggiunto un file di configurazione, ``config/packages/doctrine.yaml``, che controlla il suo comportamento. L'impostazione principale è il *DSN* (Data Source Name), una stringa contenente tutte le informazioni sulla connessione: credenziali, host, porta, ecc. Per impostazione predefinita, Doctrine cerca una variabile d'ambiente ``DATABASE_URL``.

Quasi tutti i pacchetti installati hanno una configurazione sotto la cartella ``config/packages/``. Nella maggior parte dei casi, i valori predefiniti sono stati scelti accuratamente per funzionare con le applicazioni più comuni.

Comprendere le convenzioni delle variabili d'ambiente di Symfony
----------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

È possibile definire la variabile ``DATABASE_URL`` manualmente nel file ``.env`` o nel file ``.env.local``. La ricetta del pacchetto ha inserito un valore di esempio per ``DATABASE_URL`` nel file ``.env``. Ma poiché la porta locale di PostgreSQL esposta da Docker potrebbe cambiare, tale definizione è piuttosto imprecisa. Cerchiamo una soluzione migliore.

Invece di specificare manualmente il valore di ``DATABASE_URL`` in un file, possiamo aggiungere a tutti i comandi il prefisso ``symfony``. Questo prefisso consentirà di rilevare i servizi eseguiti da Docker e/o Platform.sh (quando il tunnel è aperto) e imposterà automaticamente la variabile d'ambiente.

Docker Compose e Platform.sh funzionano perfettamente con Symfony, grazie a queste variabili d'ambiente.

.. index::
    single: Symfony CLI;var:export

Controllare tutte le variabili d'ambiente esposte eseguendo ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

Ricordate il *nome del servizio* ``database`` usato nelle configurazioni Docker e Platform.sh? I nomi dei servizi sono usati come prefissi per definire variabili d'ambiente come ``DATABASE_URL``. Se i servizi sono nominati secondo le convenzioni di Symfony, non sono necessarie altre configurazioni.

.. note::

    I database non sono l'unico servizio che beneficia delle convenzioni di Symfony. Lo stesso vale per Mailer, per esempio (tramite la variabile d'ambiente ``MAILER_DSN``).

Modifica del valore predefinito DATABASE_URL in .env
----------------------------------------------------

Modificheremo ancora il file ``.env``, cambiando il valore predefinito di ``DATABASE_URL``, affinché utilizzi PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -28,7 +28,7 @@ MESSENGER_TRANSPORT_DSN=doctrine://default?auto_setup=0
     #
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=5.7"
    -DATABASE_URL="postgresql://symfony:ChangeMe@127.0.0.1:5432/app?serverVersion=13&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Perché le informazioni devono essere duplicate in due punti diversi? Perché su alcune piattaforme Cloud, in *fase di compilazione*, l'URL del database potrebbe non essere ancora noto, ma Doctrine ha bisogno di conoscere il tipo di database (PostgreSQL, MySQL, SQLite, ecc) per costruire la sua configurazione. Quindi, l'host, il nome utente e la password non hanno molta importanza.

Creazione di classi Entity
--------------------------

Una conferenza può essere descritta con alcune proprietà:

* La *città* dove viene organizzata la conferenza;

* L' *anno* della conferenza;

* Un campo *international* per indicare se la conferenza è locale o internazionale (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

MakerBundle può aiutarci a generare una classe (una classe *Entity*) che rappresenta una conferenza.

Ẽ giunto il momento di generare l'entità ``Conference``

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Questo comando è interattivo: vi guiderà nel processo di aggiunta di tutti i campi necessari. Scegliere le seguenti risposte (la maggior parte di esse sono quelle predefinite, in modo da poterle scegliere premendo il tasto "Invio"):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Ecco l'output completo quando si esegue il comando:

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

La classe ``Conference`` è stata salvata sotto il namespace ``App\Entity\``.

Il comando ha anche generato una classe *repository* di Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Il codice generato ha il seguente aspetto (solo una piccola parte del file viene riportata):

.. code-block:: php
    :caption: src/App/Entity/Conference.php
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

Si noti che la classe stessa è una semplice classe PHP, senza segni di Doctrine. Gli attributi sono usati per aggiungere metadati utili a Doctrine per mappare la classe alla relativa tabella del database.

Doctrine ha aggiunto una proprietà ``id`` per memorizzare la chiave primaria della riga nella tabella del database. Questa chiave (``ORM\Id()``) viene generata automaticamente (``ORM\GeneratedValue()``) tramite una strategia che dipende dal tipo di database.

.. index::
    single: Command;make:entity

Ora, generiamo una classe entity per i commenti alla conferenza:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Inserire le seguenti risposte:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime_immutable``, ``no``.

Collegare le entity
-------------------

.. index::
    single: Command;make:entity

Le due entity, Conference e Comment, dovrebbero essere collegate tra loro. Una conferenza può avere zero o più commenti, che è detta relazione *uno-a-molti*.

Usare di nuovo il comando ``make:entity`` per aggiungere questa relazione alla classe ``Conference``:

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

    Se si inserisce ``?`` come risposta al tipo, si ottengono tutti i tipi supportati:

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

Date un'occhiata al diff completo per le classi entity dopo aver aggiunto la relazione:

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

Tutto ciò di cui si ha bisogno per gestire la relazione è stato generato. Una volta generato, il codice diventa dello sviluppatore, che è libero di personalizzarlo come preferisce.

Aggiungere altre proprietà
---------------------------

.. index::
    single: Command;make:entity

Ho appena realizzato che ci siamo dimenticati di aggiungere una proprietà sull'entity Comment: i partecipanti potrebbero voler allegare una foto della conferenza per illustrare il loro feedback.

Eseguire ``make:entity`` ancora una volta e aggiungere una proprietà/colonna ``photoFilename`` di tipo ``string``, ma consentendo che accetti il valore ``null`` in modo da rendere opzionale il caricamento:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrazione del database
-----------------------

.. index:: ! Command;make:migration

Il modello di progetto è ora completamente descritto dalle due classi generate.

Successivamente, abbiamo bisogno di creare le tabelle del database relative a queste entity PHP.

*Doctrine Migrations* è la soluzione perfetta per un compito del genere. È già stato installato come parte della dipendenza ``orm``.

Una *migrazione* è una classe che descrive le modifiche necessarie per aggiornare lo schema di un database dal suo stato attuale a quello nuovo, definito dagli attributi delle entity. Poiché per ora il database è vuoto, la migrazione dovrebbe essere composta dalla sola creazione di due tabelle.

Vediamo cosa genera Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

Si noti il nome del file generato nell'output (un nome che assomiglia a ``migrations/Version20191019083640.php``):

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

Aggiornamento del database locale
---------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Ora è possibile eseguire la migrazione generata per aggiornare lo schema del database locale:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Lo schema del database locale è ora aggiornato e pronto a memorizzare dati.

Aggiornamento del database di produzione
----------------------------------------

I passi necessari per migrare il database di produzione sono gli stessi visti in precedenza: commit delle modifiche e deploy.

Quando si esegue il deploy del progetto, Platform.sh aggiorna il codice, ma esegue anche l'eventuale migrazione del database (verificando la presenza del comando ``doctrine:migrations:migrate``).

.. sidebar:: Andare oltre

    * `Database e ORM di Doctrine`_ nelle applicazioni di Symfony;

    * `Tutorial su Doctrine in SymfonyCasts`_;

    * `Lavorare con Associazioni/Relazioni di Doctrine`_;

    * `Documentazione di DoctrineMigrationsBundle`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Database e ORM di Doctrine`: https://symfony.com/doc/current/doctrine.html
.. _`Tutorial su Doctrine in SymfonyCasts`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Lavorare con Associazioni/Relazioni di Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`Documentazione di DoctrineMigrationsBundle`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
