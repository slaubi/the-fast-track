Décrire la structure des données
==================================

.. index::
    single: Doctrine
    single: Database

Pour interagir avec la base de données depuis PHP, nous allons nous appuyer sur `Doctrine`_, un ensemble de bibliothèques qui nous aide à gérer les bases de données : Doctrine DBAL (une couche d'abstraction de la base de données), Doctrine ORM (une  librairie pour manipuler le contenu de notre base de données en utilisant des objets PHP), et Doctrine Migrations.

Configurer Doctrine ORM
-----------------------

.. index::
    single: Doctrine;Configuration

Comment est-ce que Doctrine est au courant de notre connexion à la base de données ? La recette de Doctrine a ajouté un fichier de configuration qui contrôle son comportement : ``config/packages/doctrine.yaml``. Le paramètre principal est le *DSN de la base de données*, une chaîne contenant toutes les informations sur la connexion : identifiants, hôte, port, etc. Par défaut, Doctrine recherche une variable d'environnement ``DATABASE_URL``.

Presque tous les paquets installés sont configurés dans le répertoire ``config/packages/``. Les valeurs par défaut ont été choisies avec soin pour fonctionner avec la plupart des applications.

Comprendre les conventions des variables d'environnement de Symfony
-------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Vous pouvez définir la variable ``DATABASE_URL`` manuellement dans le fichier ``.env`` ou ``.env.local``. En fait, grâce à la recette du paquet, vous verrez un exemple de variable ``DATABASE_URL`` dans votre fichier ``.env``. Mais comme le port exposé par Docker vers PostgreSQL peut changer, c'est assez lourd. Il y a une meilleure solution.

Au lieu de coder en dur la variable ``DATABASE_URL`` dans un fichier, nous pouvons préfixer toutes les commandes avec ``symfony``. Ceci détectera les services exécutés par Docker et/ou Platform.sh (lorsque le tunnel est ouvert) et définira automatiquement la variable d'environnement.

Docker Compose et Platform.sh fonctionnent parfaitement avec Symfony grâce à ces variables d'environnement.

.. index::
    single: Symfony CLI;var:export

Vérifiez toutes les variables d'environnement exposées en exécutant ``symfony var:export`` :

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

Vous rappelez-vous du *nom du service* ``database`` utilisé dans les configurations Docker et Platform.sh ? Les noms des services sont utilisés comme préfixes pour définir des variables d'environnement telles que ``DATABASE_URL``. Si vos services sont nommés selon les conventions Symfony, aucune autre configuration n'est nécessaire.

.. note::

    Les bases de données ne sont pas les seuls services qui bénéficient des conventions Symfony. Il en va de même pour Mailer, par exemple (via la variable d'environnement ``MAILER_DSN``).

Modifier la valeur par défaut de DATABASE_URL dans le fichier .env
-------------------------------------------------------------------

Nous allons quand même changer le fichier ``.env`` pour initialiser la variable ``DATABASE_URL`` pour l'utilisation de PostgreSQL :

.. code-block:: diff

    --- a/.env
    +++ b/.env
    @@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

Pourquoi l'information doit-elle être dupliquée à deux endroits différents ? Parce que sur certaines plates-formes de Cloud, au *moment de la compilation*, l'URL de la base de données n'est peut-être pas encore connue mais Doctrine a besoin de connaître le moteur de la base de données pour initialiser sa configuration. Ainsi, l'hôte, le pseudo et le mot de passe n'ont pas vraiment d'importance.

Créer des classes d'entités
-----------------------------

Une conférence peut être décrite en quelques propriétés :

* La *ville* où la conférence est organisée ;

* L'*année* de la conférence ;

* Une option *international* pour indiquer si la conférence est locale ou internationale (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

Le *Maker Bundle* peut nous aider à générer une classe (une classe *Entity*) qui représente une conférence.

Il est maintenant temps de générer l'entité ``Conférence`` :

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Cette commande est interactive : elle vous guidera dans le processus d'ajout de tous les champs dont vous avez besoin. Utilisez les réponses suivantes (la plupart d'entre elles sont les valeurs par défaut, vous pouvez donc appuyer sur la touche "Entrée" pour les utiliser) :

* ``city``, ``string``, ``255``, ``no`` ;
* ``year``, ``string``, ``4``, ``no`` ;
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

La commande a également généré une classe de *repository* Doctrine : ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

Le code généré ressemble à ce qui suit (seule une petite partie du fichier est retranscrite ici) :

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

Notez que la classe elle-même est une classe PHP sans aucune référence à Doctrine. Les attributs sont utilisés pour ajouter des métadonnées utiles à Doctrine afin de mapper la classe à sa table associée dans la base de données.

Doctrine a ajouté un attribut ``id`` pour stocker la clé primaire de la ligne dans la table de la base de données. Cette clé (``ORM\Id()``) est générée automatiquement (``ORM\GeneratedValue()``) avec une stratégie qui dépend du moteur de base de données.

.. index::
    single: Command;make:entity

Maintenant, générez une classe d'entité pour les commentaires de la conférence :

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

Entrez les réponses suivantes :

* ``author``, ``string``, ``255``, ``no`` ;
* ``text``, ``text``, ``no`` ;
* ``email``, ``string``, ``255``, ``no`` ;
* ``createdAt``, ``datetime_immutable``, ``no``.

Lier les entités
-----------------

.. index::
    single: Command;make:entity

Les deux entités, *Conference* et *Comment*, devraient être liées l'une à l'autre. Une conférence peut avoir zéro commentaire ou plus, ce qui s'appelle une relation *one-to-many*.

Utilisez à nouveau la commande ``make:entity`` pour ajouter cette relation à la classe ``Conference`` :

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
    single: Attributes;ORM\\ManyToOne
    single: Attributes;ORM\\JoinColumn
    single: Attributes;ORM\\OneToMany

Jetez un coup d'oeil au *diff* complet entre les classes d'entités après l'ajout de la relation :

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

Tout ce dont vous avez besoin pour gérer la relation a été généré pour vous. Une fois généré, le code devient le vôtre ; n'hésitez pas à le personnaliser comme vous le souhaitez.

Ajouter d'autres propriétés
-----------------------------

.. index::
    single: Command;make:entity

Je viens de réaliser que nous avons oublié d'ajouter une propriété sur l'entité *Comment* : une photo de la conférence peut être jointe afin d'illustrer un retour d'expérience.

Exécutez à nouveau ``make:entity`` et ajoutez une propriété/colonne ``photoFilename`` de type ``string``. Mais, comme l'ajout d'une photo est facultatif, permettez-lui d'être ``null`` :

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrer la base de données
--------------------------

.. index:: ! Command;make:migration

La structure du projet est maintenant entièrement décrite par les deux classes générées.

Ensuite, nous devons créer les tables de base de données liées à ces entités PHP.

*Doctrine Migrations* est la solution idéale pour cela. Le paquet a déjà été installé dans le cadre de la dépendance ``orm``.

Une *migration* est une classe qui décrit les changements nécessaires pour mettre à jour un schéma de base de données, de son état actuel vers le nouveau, en fonction des attributs de l'entité. Comme la base de données est vide pour l'instant, la migration devrait consister en la création de deux tables.

Voyons ce que Doctrine génère :

.. code-block:: terminal

    $ symfony console make:migration

Notez le nom du fichier généré (un nom qui ressemble à ``migrations/Version20191019083640.php``) :

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

Mettre à jour la base de données locale
-----------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Vous pouvez maintenant exécuter la migration générée pour mettre à jour le schéma de la base de données locale :

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Le schéma de la base de données locale est à jour à présent, prêt à stocker des données.

Mettre à jour la base de données de production
------------------------------------------------

Les étapes nécessaires à la migration de la base de données de production sont les mêmes que celles que vous connaissez déjà : *commiter* les changements et déployer.

Lors du déploiement du projet, Platform.sh met à jour le code, mais exécute également la migration de la base de données si nécessaire (il détecte si la commande ``doctrine:migrations:migrate`` existe).

.. sidebar:: Aller plus loin

    * `Bases de données et Doctrine ORM`_ dans les applications Symfony ;

    * `Tutoriel SymfonyCasts sur Doctrine`_ ;

    * `Travailler avec les associations/relations de Doctrine`_ ;

    * `Documentation du DoctrineMigrationsBundle`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`Bases de données et Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`Tutoriel SymfonyCasts sur Doctrine`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Travailler avec les associations/relations de Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`Documentation du DoctrineMigrationsBundle`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
