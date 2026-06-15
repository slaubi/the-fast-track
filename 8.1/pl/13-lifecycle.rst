Zarządzanie cyklem życia obiektów Doctrine
=============================================

Byłoby wspaniale, gdyby przy tworzeniu nowego komentarza atrybut ``createdAt`` przyjął automatycznie wartość bieżącej daty i godziny.

Doctrine oferuje wiele możliwości manipulowania obiektami i ich atrybutami podczas cyklu ich życia (przed utworzeniem rekordu w bazie danych, po aktualizacji rekordu, ...).

Definiowanie wywołań zwrotnych cyklu życia (ang. lifecycle callbacks)
------------------------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

Jeśli schemat działania nie wymaga dostępu do żadnej usługi i ma zastosowanie tylko do jednego rodzaju encji, należy zdefiniować wywołanie zwrotne (ang. callback) w klasie encji:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -57,8 +57,6 @@ class CommentCrudController extends AbstractCrudController
             ]);
             if (Crud::PAGE_EDIT === $pageName) {
                 yield $createdAt->setFormTypeOption('disabled', true);
    -        } else {
    -            yield $createdAt;
             }
         }
     }
    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -86,6 +87,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue(): void
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

*Zdarzenie* ``ORM\PrePersist`` jest emitowane, gdy obiekt jest po raz pierwszy zapisany w bazie danych. Gdy tak się stanie, wywołana zostanie metoda ``setCreatedAtValue()``, a jako wartość atrybutu ``createdAt`` zostanie użyta bieżąca data i czas.

Dodawanie slugów do konferencji
--------------------------------

Adresy URL konferencji nie mówią nam zbyt wiele: ``/conference/1``. Co więcej, są one zależne od szczegółów implementacji (klucz podstawowy w bazie danych jest widoczny dla użytkownika).

A gdyby tak użyć adresów URL następującej formie: ``/conference/paris-2020``? Wyglądałoby to o wiele lepiej. Fragment ``paris-2020`` jest tym, co określamy mianem *slug*.

.. index::
    single: Command;make:entity

Dodaj nowy atrybut o nazwie ``slug`` dla konferencji (pole przechowujące ciąg znaków o długości do 255 znaków, nieprzyjmujące wartości ``null``):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Utwórz plik migracji, aby dodać nową kolumnę:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

I wykonaj nowo utworzoną migrację:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Widzisz komunikat błędu? To oczekiwany efekt. Dlaczego? Ponieważ zdefiniowaliśmy kolumnę ``slug`` tak, aby nie przyjmowała wartości ``null``. Jeśli migracja zostałaby wykonana, to rekordy konferencji istniejące w bazie danych otrzymałyby wartość ``null`` w kolumnie ``slug``. Naprawmy to, modyfikując naszą migrację:

.. code-block:: diff
    :caption: patch_file

    --- i/migrations/Version00000000000000.php
    +++ w/migrations/Version00000000000000.php
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

Sztuczka polega na tym, aby najpierw dodać kolumnę i pozwolić na dodanie w niej wartości ``null``, następnie zmodyfikować istniejące rekordy ustawiając wartość ``slug`` na wartość inną niż ``null``, a na koniec zmodyfikować kolumnę ``slug``, tak aby wartość ``null`` nie była dozwolona.

.. note::

    W realnym projekcie używanie ``CONCAT(LOWER(city), '-', year)`` może okazać się niewystarczające. W takim przypadku będziemy musieli użyć "prawdziwego" Sluggera.

.. index::
    single: Command;doctrine:migrations:migrate

Tym razem migracja powinna wykonać się jak należy:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

Ponieważ aplikacja wkrótce użyje slugów do odnajdowania wymaganej konferencji, dostosujmy encję Conference aby zagwarantować, że wartości slugów będą unikalne w bazie danych:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
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
    @@ -30,7 +32,7 @@ class Conference
         #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

Jak można się było domyślać, musimy wykonać kolejne migracje:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Generowanie slugów
-------------------

.. index::
    single: Components;String
    single: Slug

Generowanie slugów, które zachowają czytelność będąc częścią adresu URL (gdzie wszystko oprócz znaków ASCII powinno być zakodowane) nie jest łatwym zadaniem, szczególnie w przypadku języków innych niż angielski. Jak przekonwertujesz na przykład ``é`` do ``e``?

Zamiast wymyślać koło na nowo, użyjmy komponentu Symfony o nazwie ``String``, który ułatwia manipulację ciągami znaków, oraz dostarcza między innymi mechanizm *sluggera*.

Dodaj w klasie ``Conference`` metodę ``computeSlug()``, której zadaniem będzie utworzenie sluga na podstawie danych konferencji:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -50,6 +51,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger): void
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

Metoda ``computeSlug()`` tworzy slug tylko wtedy, gdy aktualny slug jest pusty lub ustawiony na specjalną wartość ``-``. Po co nam ta specjalna wartość ``-``? Ponieważ przy dodawaniu konferencji w panelu administracyjnym slug jest wartością wymaganą. Potrzebujemy zatem niepustej wartości która przekaże aplikacji, że chcemy, aby slug został wygenerowany automatycznie.

Definiowanie złożonych wywołań zwrotnych cyklu życia (ang. lifecycle callback)
-----------------------------------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

Podobnie jak w przypadku atrybutu ``createdAt``, gdy konferencja zostaje zmodyfikowana, wartość atrybutu ``slug`` powinna zostać automatycznie uaktualniona poprzez wywołanie metody  ``computeSlug()``.

Ponieważ metoda zależy od implementacji ``SluggerInterface``, nie możemy w prosty sposób dodać obsługi zdarzenia ``prePersist``, jak miało to miejsce w poprzednim przypadku (nie mamy możliwości wstrzyknięcia sluggera).

Zamiast tego utwórz nasłuchiwacz zdarzeń Doctrine (ang. Doctrine entity listener):

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\PrePersistEventArgs;
    use Doctrine\ORM\Event\PreUpdateEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
        }

        public function prePersist(Conference $conference, PrePersistEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, PreUpdateEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }
    }

Zauważ, że slug jest aktualizowany w momencie, gdy tworzona jest nowa konferencja (``prePersist()``) i gdy jest ona modyfikowana (``preUpdate()``).

Konfigurowanie usługi w kontenerze
-----------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Do tej pory nie mówiliśmy o jednym z kluczowych komponentów Symfony, czyli o *kontenerze wstrzykiwania zależności* (ang. dependency injection container). Kontener jest odpowiedzialny za zarządzanie *usługami* (ang. services): ich tworzenie i wstrzykiwanie kiedy są wymagane.

*Usługa* (ang. service) jest obiektem "globalnym", który oferuje różnego rodzaju funkcje (np. mailer, logger, slugger itp.) w odróżnieniu od *obiektów danych* (np. instancje encji Doctrine).

Rzadko wchodzi się w bezpośrednią interakcję z kontenerem, ponieważ automatycznie wstrzykuje on obiekty usług kiedy są one wymagane: na przykład, sprawdzając typ argumentu kontrolera zdefiniowany przy użyciu mechanizmu podpowiadania typów  (ang. type-hinting), kontener wstrzykuje odpowiedni typ usługi.

Jeśli zastanawiało Cię, jak w poprzednim kroku został zarejestrowany nasłuchiwacz zdarzeń (ang. event listener), teraz poznasz odpowiedź: poprzez kontener. Kiedy klasa implementuje określone interfejsy, kontener wie, że klasa musi być zarejestrowana w określony sposób.

Ponieważ nasza klasa nie implementuje żadnego interfejsu ani nie rozszerza żadnej klasy bazowej, Symfony nie wie, jak ją automatycznie skonfigurować. Zamiast tego możemy użyć atrybutu, aby powiedzieć kontenerowi Symfony, jak ją podłączyć:

.. code-block:: diff
    :caption: patch_file

    --- i/src/EntityListener/ConferenceEntityListener.php
    +++ w/src/EntityListener/ConferenceEntityListener.php
    @@ -3,10 +3,14 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\PrePersistEventArgs;
     use Doctrine\ORM\Event\PreUpdateEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    Nie należy mylić nasłuchiwaczy zdarzeń Doctrine i Symfony. Mimo że wyglądają bardzo podobnie, pod spodem korzystają z osobnych mechanizmów.

Stosowanie slugów w aplikacji
------------------------------

Spróbuj dodać kilka konferencji w panelu administracyjnym i zmienić miasto lub rok jednej z nich. Slug nie zostanie w tym przypadku zaktualizowany, chyba że użyjesz specjalnej wartości ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

Ostatnią zmianą jest aktualizacja kontrolerów i szablonów, tak aby mechanizm routingu wykorzystywał ``slug`` zamiast ``id`` konferencji. Ponieważ parametr trasy nie jest już kluczem podstawowym encji, wskaż atrybutowi ``#[MapEntity]``, którą właściwość ma dopasować, przekazując jawne ``mapping``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -20,7 +20,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    #[Route('/conference/{slug}', name: 'conference')]
    +    public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -16,7 +16,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
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

Dostęp do stron konferencji powinien być teraz możliwy z wykorzystaniem sluga:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Idąc dalej

    * `System zdarzeń Doctrine`_ (lifecycle callbacks and listeners, entity listeners and lifecycle subscribers);

    * `Dokumentacja komponentu String`_;

    * `Kontener usług`_;

    * `Ściągawka Symfony Services`_.

.. _`System zdarzeń Doctrine`: https://symfony.com/doc/current/doctrine/events.html
.. _`Dokumentacja komponentu String`: https://symfony.com/doc/current/components/string.html
.. _`Kontener usług`: https://symfony.com/doc/current/service_container.html
.. _`Ściągawka Symfony Services`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
