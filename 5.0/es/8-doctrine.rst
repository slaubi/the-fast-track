Describiendo la estructura de datos
===================================

.. index::
    single: Doctrine
    single: Database

Para trabajar con la base de datos desde PHP, vamos a hacer uso de `Doctrine`_, un conjunto de librerías que ayudan a los desarrolladores a gestionar bases de datos:

.. code-block:: bash

    $ symfony composer req "orm:^2"

Este comando instala algunas dependencias: Doctrine DBAL (una capa de abstracción de base de datos), Doctrine ORM (una biblioteca para manipular el contenido de nuestra base de datos usando objetos PHP), y Doctrine Migrations.

Configurando Doctrine ORM
-------------------------

.. index::
    single: Doctrine;Configuration

¿De dónde obtiene Doctrine los datos de conexión con la base de datos? La receta de Doctrine agregó un archivo de configuración, ``config/packages/doctrine.yaml``, que controla su comportamiento. La configuración principal es el *DSN de la base de datos*, una cadena que contiene toda la información sobre la conexión: credenciales, host, puerto, etc. Por defecto, Doctrine busca una variable de entorno ``DATABASE_URL``.

Entendiendo las convenciones de las variables de entorno de Symfony
-------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Puedes definir ``DATABASE_URL`` manualmente en el archivo ``.env`` o ``.env.local``. De hecho, gracias a la receta del paquete, verás un ejemplo  de ``DATABASE_URL`` en tu archivo ``.env``. Pero debido a que el puerto local a PostgreSQL expuesto por Docker puede cambiar, es bastante engorroso. Hay una manera mejor.

En lugar de la definir ``DATABASE_URL`` en un archivo, podemos prefijar todos los comandos con ``symfony``. Esto detectará los servicios ejecutados por Docker y/o SymfonyCloud (cuando el túnel está abierto) y establecerá la variable de entorno automáticamente.

Docker Compose y SymfonyCloud funcionan perfectamente con Symfony gracias a estas variables de entorno.

.. index::
    single: Symfony CLI;var:export

Comprueba todas las variables de entorno expuestas ejecutando ``symfony var:export``:

.. code-block:: bash

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://main:main@127.0.0.1:32781/main?sslmode=disable&charset=utf8
    # ...

¿Recuerdas el *nombre del servicio* ``database`` utilizado en las configuraciones Docker y SymfonyCloud? Los nombres de servicio se utilizan como prefijos para definir variables de entorno como ``DATABASE_URL``. Si tus servicios se nombran de acuerdo a las convenciones de Symfony, no se necesita ninguna otra configuración.

.. note::

    Las bases de datos no son el único servicio que se beneficia de las convenciones de Symfony. Lo mismo ocurre con Mailer, por ejemplo (mediante la variable de entorno ``MAILER_DSN``).

Cambiando el valor por defecto de DATABASE_URL en .env
------------------------------------------------------

Aún así cambiaremos el archivo ``.env`` para configurar el valor predeterminado ``DATABASE_DSN`` para usar PostgreSQL:

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

¿Por qué es necesario duplicar la información en dos lugares diferentes? Porque en algunas plataformas Cloud, a la *hora de crear la aplicación*, puede que no se conozca todavía la URL de la base de datos, pero Doctrine necesita conocer el motor de la base de datos para construir su configuración. Por lo tanto, el host, el nombre de usuario y la contraseña no importan realmente.

Creando clases de entidad (Entities)
------------------------------------

Una conferencia puede describirse con algunas propiedades:

* La *ciudad* donde se organiza la conferencia;

* El *año* de la conferencia;

* Una valor booleano *internacional* para indicar si la conferencia es local o internacional (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

El *bundle* Maker puede ayudarnos a generar una clase (una clase *Entity*) que representa una conferencia:

.. code-block:: bash
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

Este comando es interactivo: te guiará a través del proceso de añadir todos los campos que necesites. Utiliza las respuestas que mostramos a continuación (la mayoría de ellas son las predeterminadas, por lo que puedes pulsar la tecla "Intro" para utilizarlas):

* ``city`` , ``string`` , ``255``, ``no``;
* ``year``, ``string``, ``4``, ``no``;
* ``isInternational``, ``boolean``, ``no``.

Aquí está la salida completa cuando se ejecuta el comando:

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

La clase ``Conference`` se ha almacenado bajo el espacio de nombres ``App\Entity\``.

El comando también generó una clase *repositorio* de Doctrine: ``App\Repository\ConferenceRepository``.

.. index::
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\Id
    single: Annotations;@ORM\\GeneratedValue
    single: Annotations;@ORM\\Column

El código generado tiene el siguiente aspecto (solo mostraremos una pequeña parte del archivo):

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

Ten en cuenta que la clase en sí es una clase PHP sencilla sin rastro de Doctrine. Las anotaciones se utilizan para añadir metadatos útiles para que Doctrine asocie la clase a la tabla correspondiente de la base de datos.

Doctrine agregó una propiedad ``id`` para almacenar la clave primaria de la fila en la tabla de la base de datos. Esta clave (``@ORM\Id()``) se crea automáticamente (``@ORM\GeneratedValue()``) mediante una estrategia que depende del motor de la base de datos.

.. index::
    single: Command;make:entity

A continuación, genera una clase Entity para los comentarios de la conferencia:

.. code-block:: bash
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime||no)

    $ symfony console make:entity Comment

Introduce las siguientes respuestas:

* ``author``, ``string``, ``255``, ``no``;
* ``text``, ``text``, ``no``;
* ``email``, ``string``, ``255``, ``no``;
* ``createdAt``, ``datetime``, ``no``.

Enlazando entidades
-------------------

.. index::
    single: Command;make:entity

Las dos entidades, Conference y Comment, deben estar vinculadas entre sí. Una conferencia puede tener cero o más comentarios, lo que se llama una relación de *uno a muchos*.

Usa el comando ``make:entity`` de nuevo para agregar esta relación a la  clase ``Conference``:

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

    Si escribes ``?`` como respuesta para el tipo, obtendrás todos los tipos soportados:

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

Echa un vistazo a los cambios que se aplican en las clases de entidad después de añadir la relación:

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

Se ha generado por ti todo lo que necesitas para gestionar la relación. A partir de ahora, el código es tuyo; no dudes en personalizarlo como quieras.

Añadiendo más propiedades
---------------------------

.. index::
    single: Command;make:entity

Acabo de darme cuenta de que nos hemos olvidado de añadir una propiedad a la entidad de comentarios: los asistentes pueden adjuntar una foto de la conferencia para ilustrar sus comentarios.

Ejecuta ``make:entity`` una vez más y añade una propiedad/columna ``photoFilename`` de tipo ``string``, pero permite que pueda ser ``null``, ya que subir una foto es opcional:

.. code-block:: bash
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrando la base de datos
-------------------------

.. index:: ! Command;make:migration

El modelo de proyecto está ahora completamente descrito por las dos clases generadas.

A continuación, necesitamos crear las tablas de base de datos relacionadas con estas entidades PHP.

*Doctrine Migrations* es el complemento perfecto para esta tarea. Ya se ha instalado como parte de la dependencia ``orm``.

Una *migración* es una clase que describe los cambios necesarios para actualizar un esquema de base de datos desde su estado actual al nuevo definido por las anotaciones de la entidad. Como la base de datos está vacía por ahora, la migración debería consistir en dos creaciones de tablas.

Veamos qué genera Doctrine:

.. code-block:: bash

    $ symfony console make:migration

Observa el nombre del archivo generado en la salida (un nombre similar a ``migrations/Version20191019083640.php``):

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

Actualizando la base de datos local
-----------------------------------

.. index:: ! Command;doctrine:migrations:migrate

Ahora puedes ejecutar la migración generada para actualizar el esquema de la base de datos local:

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

El esquema de la base de datos local ya está actualizado y listo para almacenar algunos datos.

Actualizando la base de datos de producción
--------------------------------------------

Los pasos necesarios para migrar la base de datos de producción son los mismos con los que ya estás familiarizado: confirmar los cambios y desplegar.

Al desplegar el proyecto, SymfonyCloud actualiza el código, pero también ejecuta la migración de la base de datos si la hay (detecta si el comando ``doctrine:migrations:migrate`` existe).

.. sidebar:: Yendo más allá

    * `Bases de datos y Doctrine ORM <https://symfony.com/doc/current/doctrine.html>`_ en aplicaciones Symfony;

    * `Tutorial de Doctrine SymfonyCasts <https://symfonycasts.com/screencast/symfony-doctrine/install>`_ ;

    * `Trabajar con Asociaciones/Relaciones de Doctrine <https://symfony.com/doc/current/doctrine/associations.html>`_ ;

    * `DoctrineMigrationsBundle docs <https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html>`_.

.. _`Doctrine`: https://www.doctrine-project.org/
