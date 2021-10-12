Décrire la structure des données
==================================

.. index::
    single: Doctrine
    single: Database

To deal with the database from PHP, we are going to depend on `Doctrine`_, a
set of libraries that help developers manage databases:

.. code-block:: bash

    $ symfony composer req "orm:^2"

This command installs a few dependencies: Doctrine DBAL (a database abstraction
layer), Doctrine ORM (a library to manipulate our database content using PHP objects),
and Doctrine Migrations.

Configurer Doctrine ORM
-----------------------

.. index::
    single: Doctrine;Configuration

How does Doctrine know the database connection? Doctrine's recipe added a
configuration file, ``config/packages/doctrine.yaml``, that controls its
behavior. The main setting is the *database DSN*, a string containing all the
information about the connection: credentials, host, port, etc. By default,
Doctrine looks for a ``DATABASE_URL`` environment variable.

Almost all installed packages have a configuration under the
``config/packages/`` directory. Most of the time, the defaults have been chosen
carefully to work for most applications.

Comprendre les conventions des variables d'environnement de Symfony
-------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

You can define the ``DATABASE_URL`` manually in the ``.env`` or ``.env.local``
file. In fact, thanks to the package's recipe, you'll see an example
``DATABASE_URL`` in your ``.env`` file. But because the local port to
PostgreSQL exposed by Docker can change, it is quite cumbersome. There is a
better way.

Instead of hard-coding ``DATABASE_URL`` in a file, we can prefix all commands
with ``symfony``. This will detect services ran by Docker and/or SymfonyCloud
(when the tunnel is open) and set the environment variable automatically.

Docker Compose and SymfonyCloud work seamlessly with Symfony thanks to
these environment variables.

.. index::
    single: Symfony CLI;var:export

Vérifiez toutes les variables d'environnement exposées en exécutant ``symfony var:export`` :

.. code-block:: bash

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

Remember the ``database`` *service name* used in the Docker and SymfonyCloud
configurations? The service names are used as prefixes to define environment
variables like ``DATABASE_URL``. If your services are named according to the
Symfony conventions, no other configuration is needed.

.. note::

    Databases are not the only service that benefit from the Symfony
    conventions. The same goes for Mailer, for example (via the ``MAILER_DSN``
    environment variable).

Modifier la valeur par défaut de DATABASE_URL dans le fichier .env
-------------------------------------------------------------------

We will still change the ``.env`` file to setup the default ``DATABASE_URL`` to
use PostgreSQL:

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -24,5 +24,5 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     #
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=5.7"
    -DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=13&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=13&charset=utf8"
     ###< doctrine/doctrine-bundle ###

Why does the information need to be duplicated in two different places? Because
on some Cloud platforms, at *build time*, the database URL might not be known
yet but Doctrine needs to know the database's engine to build its
configuration. So, the host, username, and password do not really matter.

Créer des classes d'entités
-----------------------------

Une conférence peut être décrite en quelques propriétés :

* The *city* where the conference is organized;

* The *year* of the conference;

* An *international* flag to indicate if the conference is local or
  international (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

The Maker bundle can help us generate a class (an *Entity* class) that
represents a conference:

.. code-block:: bash
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

This command is interactive: it will guide you through the process of adding
all the fields you need. Use the following answers (most of them are
the defaults, so you can hit the "Enter" key to use them):

* ``city``, ``string``, ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Voici la sortie complète lors de l'exécution de la commande :

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

La classe ``Conference`` a été stockée sous le *namespace* ``App\Entity\``.

The command also generated a Doctrine *repository* class:
``App\Repository\ConferenceRepository``.

.. index::
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\Id
    single: Annotations;@ORM\\GeneratedValue
    single: Annotations;@ORM\\Column

The generated code looks like the following (only a small portion of the file
is replicated here):

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

Note that the class itself is a plain PHP class with no signs of Doctrine.
Annotations are used to add metadata useful for Doctrine to map the class to
its related database table.

Doctrine added an ``id`` property to store the primary key of the row in
the database table. This key (``@ORM\Id()``) is automatically generated
(``@ORM\GeneratedValue()``) via a strategy that depends on the database engine.

.. index::
    single: Command;make:entity

Maintenant, générez une classe d'entité pour les commentaires de la conférence :

.. code-block:: bash
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime||no)

    $ symfony console make:entity Comment

Entrez les réponses suivantes :

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime``, ``no``.

Lier les entités
-----------------

.. index::
    single: Command;make:entity

The two entities, Conference and Comment, should be linked together. A
Conference can have zero or more Comments, which is called a *one-to-many*
relationship.

Use the ``make:entity`` command again to add this relationship to the
``Conference`` class:

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

    Si vous entrez ``?`` comme réponse pour le type, vous obtiendrez tous les types pris en charge :

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

Have a look at the full diff for the entity classes after adding the
relationship:

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

Everything you need to manage the relationship has been generated for you. Once
generated, the code becomes yours; feel free to customize it the way you want.

Ajouter d'autres propriétés
-----------------------------

.. index::
    single: Command;make:entity

I just realized that we have forgotten to add one property on the Comment
entity: attendees might want to attach a photo of the conference to illustrate
their feedback.

Run ``make:entity`` once more and add a ``photoFilename`` property/column of
type ``string``, but allow it to be ``null`` as uploading a photo is optional:

.. code-block:: bash
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrer la base de données
--------------------------

.. index:: ! Command;make:migration

La structure du projet est maintenant entièrement décrite par les deux classes générées.

Ensuite, nous devons créer les tables de base de données liées à ces entités PHP.

*Doctrine Migrations* is the perfect match for such a task. It has already been
installed as part of the ``orm`` dependency.

A *migration* is a class that describes the changes needed to update a
database schema from its current state to the new one defined by the
entity annotations. As the database is empty for now, the migration should
consist of two table creations.

Voyons ce que Doctrine génère :

.. code-block:: bash

    $ symfony console make:migration

Notice the generated file name in the output (a name that looks like
``migrations/Version20191019083640.php``):

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

Mettre à jour la base de données locale
-----------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Vous pouvez maintenant exécuter la migration générée pour mettre à jour le schéma de la base de données locale :

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Le schéma de la base de données locale est à jour à présent, prêt à stocker des données.

Mettre à jour la base de données de production
------------------------------------------------

The steps needed to migrate the production database are the same as the ones
you are already familiar with: commit the changes and deploy.

When deploying the project, SymfonyCloud updates the code, but also runs the
database migration if any (it detects if the ``doctrine:migrations:migrate``
command exists).

.. sidebar:: Going Further

    * `Databases and Doctrine ORM
      <https://symfony.com/doc/current/doctrine.html>`_ in Symfony applications;

    * `SymfonyCasts Doctrine tutorial
      <https://symfonycasts.com/screencast/symfony-doctrine/install>`_;

    * `Working with Doctrine Associations/Relations
      <https://symfony.com/doc/current/doctrine/associations.html>`_;

    * `DoctrineMigrationsBundle docs
      <https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html>`_.

.. _`Doctrine`: https://www.doctrine-project.org/
