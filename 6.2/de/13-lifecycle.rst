Den Lifecycle von Doctrine-Objekten verwalten
=============================================

Beim Erstellen eines neuen Kommentars wäre es gut, wenn das ``createdAt``-Datum automatisch auf das aktuelle Datum und die aktuelle Uhrzeit gesetzt würde.

Doctrine hat verschiedene Möglichkeiten, Objekte und deren Properties (Eigenschaften) während ihres Lifecycle zu manipulieren (bevor die Zeile in der Datenbank erstellt wird, nachdem die Zeile aktualisiert wird, ...).

Lifecycle-Callbacks definieren
------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

Wenn das Verhalten nicht von einem Service abhängt und nur auf eine bestimmte Entity angewendet werden soll, definierst Du einen Callback in der Entity-Klasse:

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

Das ``ORM\PrePersist``-*Event* wird ausgelöst, wenn das Objekt zum ersten Mal in der Datenbank gespeichert wird. In diesem Fall wird die ``setCreatedAtValue()``-Methode aufgerufen und das aktuelle Datum und die aktuelle Uhrzeit für den Wert der ``createdAt``-Property/Spalte verwendet.

Slugs zu Konferenzen hinzufügen
--------------------------------

Die URLs für Konferenzen sind nicht aussagekräftig: ``/conference/1``. Noch wichtiger ist, dass sie von einem Implementierungsdetail abhängen (der Primärschlüssel der Datenbank wird veröffentlicht).

Wie sieht es mit der Verwendung von URLs wie ``/conference/paris-2020`` aus? Das würde viel besser aussehen. Wir nennen ``paris-2020`` den Konferenz-*Slug*.

.. index::
    single: Command;make:entity

Füge ein neues ``slug``-Property für Konferenzen hinzu (eine Zeichenkette mit 255 Zeichen, die nicht leer sein darf):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Erstelle eine Migration, um die neue Spalte hinzuzufügen:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

Und führe diese neue Migration aus:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Bekommst Du einen Fehler? Das war zu erwarten. Warum? Weil wir festgelegt haben, dass der Slug nicht ``null`` (leer) sein darf, aber bestehende Einträge in der Konferenzdatenbank werden beim Ausführen der Migration einen ``null``-Wert erhalten. Lass uns das beheben, indem wir die Migration verbessern:

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

Der Trick hier ist, die Spalte hinzuzufügen und dabei ``null``-Werte zuzulassen, anschließend den Slug zu setzen und schließlich die Slug-Spalte so zu ändern, dass sie ``null`` nicht erlaubt.

.. note::

    Für ein echtes Projekt ist die Verwendung ``CONCAT(LOWER(city), '-', year)`` möglicherweise nicht ausreichend. In diesem Fall müssten wir den "echten" Slugger verwenden.

.. index::
    single: Command;doctrine:migrations:migrate

Die Migration sollte jetzt fehlerfrei laufen:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

Da die Anwendung bald Slugs verwenden wird, um jede Konferenz zu finden, sollten wir die Konferenz-Entity verbessern, um sicherzustellen, dass die Slug-Werte in der Datenbank eindeutig sind:

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

Wie du vielleicht schon erraten hast, müssen wir den Migrationstanz aufführen:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Slugs generieren
----------------

.. index::
    single: Components;String
    single: Slug

Das Erzeugen eines Slug, der in einer URL gut lesbar ist (wobei alles außer ASCII-Zeichen kodiert werden sollte), ist eine schwierige Aufgabe, insbesondere für andere Sprachen als Englisch. Wie konvertiert man ``é`` zum Beispiel zu ``e``?

Anstatt das Rad neu zu erfinden, verwenden wir die Symfony-Komponente ``String``, die die Manipulation von Zeichenketten erleichtert und einen *Slugger* bietet.

Füge in der ``Conference``-Klasse eine ``computeSlug()``-Methode hinzu, die den Slug basierend auf den Konferenzdaten erstellt:

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

Die ``computeSlug()``-Methode erstellt einen Slug nur, wenn der aktuelle Slug leer ist oder auf den speziellen ``-``-Wert eingestellt ist. Warum brauchen wir den besonderen ``-``-Wert? Beim Hinzufügen einer Konferenz im Backend wird der Slug benötigt. Wir benötigen also einen nicht-leeren Wert, der der Anwendung mitteilt, dass wird den Slug automatisch generieren lassen möchten.

Einen komplexen Lifecycle-Callback definieren
---------------------------------------------

.. index::
    single: Doctrine;Entity Listener

Wie bei der ``createdAt``-Property, soll der ``slug` jedesmal automatisch durch den Aufruf der ``computeSlug()``-Methode aktualisiert werden, wenn die Konferenz geändert wird.

Da diese Methode jedoch von einer ``SluggerInterface``-Implementierung abhängt, können wir kein ``prePersist``-Event wie bisher hinzufügen (wir haben keine Möglichkeit, den Slugger zu injizieren).

Erstelle stattdessen einen Doctrine Entity Listener:

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

Beachte, dass der Slug aktualisiert wird, wenn eine neue Konferenz erstellt wird (``prePersist()``) und wenn sie aktualisiert wird (``preUpdate()``).

Einen Service im Container konfigurieren
----------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Bisher haben wir noch nicht über eine Schlüsselkomponente von Symfony gesprochen, den *Dependency Injection Container.* Der Container ist für die Verwaltung der *Services* verantwortlich: Er erstellt und injiziert sie bei Bedarf.

Ein *Service* ist ein "globales" Objekt, das Funktionen bereitstellt, z. B. einen Mailer, einen Logger, einen Slugger, etc. (im Gegensatz zu *Datenobjekten* wie z. B. Doctrine Entity Instanzen).

Du interagierst selten direkt mit dem Container, da er automatisch Service-Objekte injiziert, wann immer Du sie benötigst: Der Container injiziert beispielsweise die Controller-Objektargumente, wenn Du sie mit Type-Hints (Typen-Hinweise) deklarierst.

Wenn Du dich gefragt hast, wie der Event-Listener im vorherigen Schritt registriert wurde, hast Du nun die Antwort: der Container. Wenn eine Klasse bestimmte Interfaces implementiert, weiß der Container, dass die Klasse auf eine bestimmte Weise registriert werden muss.

Hier, weil unsere Klasse weder Interfaces implementiert, noch irgendeine Basisklasse erweitert, weiss Symfony nicht, wie er automatisch konfiguriert wird:

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

    Verwechsel die Listener von Doctrine Events nicht mit denen von Symfony. Auch wenn sie sehr ähnlich aussehen, nutzen sie unter der Haube nicht die gleiche Infrastruktur.

Slugs in der Anwendung nutzen
-----------------------------

Versuche, weitere Konferenzen im Backend hinzuzufügen und ändere  die Stadt oder das Jahr einer bestehenden Konferenz; der Slug wird nicht aktualisiert, es sei denn Du verwendest den speziellen ``-``-Wert.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

Die letzte Änderung besteht darin, die Controller und die Templates zu anzupassen, sodass diese den Konferenz-``slug`` anstelle der Konferenz-``id`` für Routen verwenden:

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

Der Zugriff auf die Konferenzseiten sollte nun über den Slug erfolgen:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Weiterführendes

    * Das `Doctrine Eventsystem`_ (Lifecycle Callbacks und Listener, Entity Listener und Lifecycle Subscriber);

    * Die `String-Komponenten-Dokumentation`_;

    * Der `Service-Container`_;

    * Das `Symfony Services Cheat Sheet`_.

.. _`Doctrine Eventsystem`: https://symfony.com/doc/current/doctrine/events.html
.. _`String-Komponenten-Dokumentation`: https://symfony.com/doc/current/components/string.html
.. _`Service-Container`: https://symfony.com/doc/current/service_container.html
.. _`Symfony Services Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
