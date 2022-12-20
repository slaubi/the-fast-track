Managing the Lifecycle of Doctrine Objects
==========================================

When creating a new comment, it would be great if the ``createdAt`` date would be set automatically to the current date and time.

Doctrine has different ways to manipulate objects and their properties during their lifecycle (before the row in the database is created, after the row is updated, ...).

Defining Lifecycle Callbacks
----------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

When the behavior does not need any service and should be applied to only one kind of entity, define a callback in the entity class:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -91,6 +92,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

The ``ORM\PrePersist`` *event* is triggered when the object is stored in the database for the very first time. When that happens, the ``setCreatedAtValue()`` method is called and the current date and time is used for the value of the ``createdAt`` property.

Adding Slugs to Conferences
---------------------------

The URLs for conferences are not meaningful: ``/conference/1``. More importantly, they depend on an implementation detail (the primary key in the database is leaked).

What about using URLs like ``/conference/paris-2020`` instead? That would look much better. ``paris-2020`` is what we call the conference *slug*.

.. index::
    single: Command;make:entity

Add a new ``slug`` property for conferences (a not nullable string of 255 characters):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Create a migration file to add the new column:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

And execute that new migration:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Got an error? This is expected. Why? Because we asked for the slug to be not ``null`` but existing entries in the conference database will get a ``null`` value when the migration is ran. Let's fix that by tweaking the migration:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema): void

The trick here is to add the column and allow it to be ``null``, then set the slug to a not ``null`` value, and finally, change the slug column to not allow ``null``.

.. note::

    For a real project, using ``CONCAT(LOWER(city), '-', year)`` might not be enough. In that case, we would need to use the "real" Slugger.

.. index::
    single: Command;doctrine:migrations:migrate

Migration should run fine now:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

Because the application will soon use slugs to find each conference, let's tweak the Conference entity to ensure that slug values are unique in the database:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,8 +6,10 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    +#[UniqueEntity('slug')]
     class Conference
     {
         #[ORM\Id]
    @@ -27,7 +29,7 @@ class Conference
         #[ORM\OneToMany(mappedBy: 'conference', targetEntity: Comment::class, orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(type: 'string', length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

As you might have guessed, we need to perform the migration dance:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Generating Slugs
----------------

.. index::
    single: Components;String
    single: Slug

Generating a slug that reads well in a URL (where anything besides ASCII characters should be encoded) is a challenging task, especially for languages other than English. How do you convert ``é`` to ``e`` for instance?

Instead of reinventing the wheel, let's use the Symfony ``String`` component, which eases the manipulation of strings and provides a *slugger*.

Add a ``computeSlug()`` method to the ``Conference`` class that computes the slug based on the conference data:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -47,6 +48,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger)
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

The ``computeSlug()`` method only computes a slug when the current slug is empty or set to the special ``-`` value. Why do we need the ``-`` special value? Because when adding a conference in the backend, the slug is required. So, we need a non-empty value that tells the application that we want the slug to be automatically generated.

Defining a Complex Lifecycle Callback
-------------------------------------

.. index::
    single: Doctrine;Entity Listener

As for the ``createdAt`` property, the ``slug`` one should be set automatically whenever the conference is updated by calling the ``computeSlug()`` method.

But as this method depends on a ``SluggerInterface`` implementation, we cannot add a ``prePersist`` event as before (we don't have a way to inject the slugger).

Instead, create a Doctrine entity listener:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
        }

        public function prePersist(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, LifecycleEventArgs $event)
        {
            $conference->computeSlug($this->slugger);
        }
    }

Note that the slug is updated when a new conference is created (``prePersist()``) and whenever it is updated (``preUpdate()``).

Configuring a Service in the Container
--------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Up until now, we have not talked about one key component of Symfony, the *dependency injection container*. The container is responsible for managing *services*: creating them and injecting them whenever needed.

A *service* is a "global" object that provides features (e.g. a mailer, a logger, a slugger, etc.) unlike *data objects* (e.g. Doctrine entity instances).

You rarely interact with the container directly as it automatically injects service objects whenever you need them: the container injects the controller argument objects when you type-hint them for instance.

If you wondered how the event listener was registered in the previous step, you now have the answer: the container. When a class implements some specific interfaces, the container knows that the class needs to be registered in a certain way.

Here, because our class doesn't implement any interface nor doesn't extend any base class, Symfony doesn't know how to auto-configure it. Instead, we can use an attribute to tell the Symfony container how to wire it:

.. code-block:: diff
    :caption: patch_file

    --- a/src/EntityListener/ConferenceEntityListener.php
    +++ b/src/EntityListener/ConferenceEntityListener.php
    @@ -3,9 +3,13 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\LifecycleEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    Don't confuse Doctrine event listeners and Symfony ones. Even if they look very similar, they are not using the same infrastructure under the hood.

Using Slugs in the Application
------------------------------

Try adding more conferences in the backend and change the city or the year of an existing one; the slug won't be updated except if you use the special ``-`` value.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

The last change is to update the controllers and the templates to use the conference ``slug`` instead of the conference ``id`` for routes:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -20,7 +20,7 @@ class ConferenceController extends AbstractController
             ]);
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    +    #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -18,7 +18,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -22,10 +22,10 @@
             {% endfor %}

             {% if previous >= 0 %}
    -            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
             {% endif %}
             {% if next < comments|length %}
    -            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
             {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>

Accessing conference pages should now be done via its slug:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Going Further

    * The `Doctrine event system`_ (lifecycle callbacks and listeners, entity listeners and lifecycle subscribers);

    * The `String component docs`_;

    * The `Service container`_;

    * The `Symfony Services Cheat Sheet`_.

.. _`Doctrine event system`: https://symfony.com/doc/current/doctrine/events.html
.. _`String component docs`: https://symfony.com/doc/current/components/string.html
.. _`Service container`: https://symfony.com/doc/current/service_container.html
.. _`Symfony Services Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
