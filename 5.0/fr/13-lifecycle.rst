Gérer le cycle de vie des objets Doctrine
==========================================

Lors de la création d'un nouveau commentaire, ce serait bien si la date ``createdAt`` était automatiquement définie à la date et à l'heure courantes.

Doctrine a différentes façons de manipuler les objets et leurs propriétés pendant leur cycle de vie (avant la création de la ligne dans la base de données, après la mise à jour de la ligne, etc.).

Définir des *lifecycle callbacks*
----------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Annotations;@ORM\\Entity
    single: Annotations;@ORM\\HasLifecycleCallbacks
    single: Annotations;@ORM\\PrePersist

Lorsque le comportement n'a besoin d'aucun service et ne doit être appliqué qu'à un seul type d'entité, définissez un callback dans la classe entité :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\ORM\Mapping as ORM;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
    + * @ORM\HasLifecycleCallbacks()
      */
     class Comment
     {
    @@ -106,6 +107,14 @@ class Comment
             return $this;
         }

    +    /**
    +     * @ORM\PrePersist
    +     */
    +    public function setCreatedAtValue()
    +    {
    +        $this->createdAt = new \DateTime();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

L'*événement* ``@ORM\PrePersist`` est déclenché lorsque l'objet est enregistré dans la base de données pour la toute première fois. Lorsque cela se produit, la méthode ``setCreatedAtValue()`` est appelée et la date et l'heure courantes sont utilisées pour la valeur de la propriété ``createdAt``.

Ajouter des *slugs* aux conférences
------------------------------------

Les URLs des conférences n'ont pas de sens : ``/conference/1``. Plus important encore, ils dépendent d'un détail d'implémentation (la clé primaire de la base de données est révélée).

Pourquoi ne pas plutôt utiliser des URLs telles que ``/conference/paris-2020`` ? Ce serait plus joli. ``paris-2020``, c'est ce que l'on appelle le *slug* de la conférence.

.. index::
    single: Command;make:entity

Ajoutez une nouvelle propriété ``slug`` pour les conférences (une chaîne non nulle de 255 caractères) :

.. code-block:: bash
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

Créez un fichier de migration pour ajouter la nouvelle colonne :

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

Et exécutez cette nouvelle migration :

.. code-block:: bash
    :class: ignore

    $ symfony console doctrine:migrations:migrate

Vous avez une erreur ? C'était prévu. Pourquoi ? Parce que nous avons demandé que le slug ne soit pas ``null``, et que les entrées existantes dans la base de données de la conférence obtiendront une valeur ``null`` lorsque la migration sera exécutée. Corrigeons cela en ajustant la migration :

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version20200714152808 extends AbstractMigration
         public function up(Schema $schema) : void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema) : void

L'astuce ici est d'ajouter la colonne et de lui permettre d'être ``null``, puis de définir une valeur non ``null`` pour le slug, et enfin, de changer la colonne de slug pour ne plus permettre ``null``.

.. note::

    Pour un projet réel, l'utilisation de ``CONCAT(LOWER(city), '-', year)`` peut ne pas suffire. Nous aurions alors besoin d'utiliser le "vrai" Slugger.

.. index::
    single: Command;doctrine:migrations:migrate

La migration devrait fonctionner maintenant :

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Annotations;@ORM\\UniqueEntity
    single: Annotations;@ORM\\Column

Étant donné que l'application utilisera bientôt les slugs pour trouver chaque conférence, ajustons l'entité Conference pour s'assurer que les valeurs des slugs soient uniques dans la base de données :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -6,9 +6,11 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    + * @UniqueEntity("slug")
      */
     class Conference
     {
    @@ -40,7 +42,7 @@ class Conference
         private $comments;

         /**
    -     * @ORM\Column(type="string", length=255)
    +     * @ORM\Column(type="string", length=255, unique=true)
          */
         private $slug;

.. index::
    single: Command;make:migration

Comme vous l'aurez deviné, nous devons exécuter la danse de la migration :

.. code-block:: bash

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: bash
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Générer des slugs
-------------------

.. index::
    single: Components;String
    single: Slug

Générer un slug qui se lit bien dans une URL (où tout ce qui n'est pas des caractères ASCII doit être encodé) est une tâche difficile, surtout pour les langues autres que l'anglais. Comment convertir ``é`` en ``e`` par exemple ?

Au lieu de réinventer la roue, utilisons le composant Symfony ``String``, qui facilite la manipulation des chaînes et fournit un slugger :

.. code-block:: bash

    $ symfony composer req string

Dans la classe ``Conference``, ajoutez une méthode ``computeSlug()``, qui calcule le slug en fonction des données de la conférence :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
    @@ -61,6 +62,13 @@ class Conference
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

La méthode ``computeSlug()`` ne calcule un slug que lorsque le slug courant est vide ou défini à la valeur spéciale ``-``. Pourquoi avons-nous besoin de cette valeur particulière ``-`` ? Parce que lors de l'ajout d'une conférence dans l'interface d'administration, le slug est nécessaire. Nous avons donc besoin d'une valeur non vide qui indique à l'application que nous voulons que le slug soit généré automatiquement.

Définir un *lifecycle callback* complexe
-----------------------------------------

.. index::
    single: Doctrine;Entity Listener

Comme pour la propriété ``createdAt``, la propriété ``slug`` doit être définie automatiquement à chaque fois que la conférence est mise à jour en appelant la méthode ``computeSlug()``.

Mais comme cette méthode dépend d'une implémentation de ``SluggerInterface``, nous ne pouvons pas ajouter un événement ``prePersist`` comme avant (nous n'avons pas la possibilité d'injecter le slugger).

Créez plutôt un listener d'entité Doctrine :

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\LifecycleEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        private $slugger;

        public function __construct(SluggerInterface $slugger)
        {
            $this->slugger = $slugger;
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

Notez que le slug est modifié lorsqu'une nouvelle conférence est créée (``prePersist()``) et lorsqu'elle est mise à jour (``preUpdate()``).

Configurer un service dans le conteneur
---------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

Jusqu'à présent, nous n'avons pas parlé d'un élément clé de Symfony, le *conteneur d'injection de dépendance*. Le conteneur est responsable de la gestion des *services* : leur création, et leur injection en cas de besoin.

Un *service* est un objet "global" qui fournit des fonctionnalités (par exemple un *mailer*, un *logger*, un *slugger*, etc.) contrairement aux *objets de données* (par exemple les instances d'entités Doctrine).

Vous interagissez rarement directement avec le conteneur car il injecte automatiquement des objets de service quand vous en avez besoin : par exemple, le conteneur injecte les objets en arguments du contrôleur lorsque vous les typez.

Si vous vous demandez comment le listener d'événement a été initialisé à l'étape précédente, vous avez maintenant la réponse : le conteneur. Lorsqu'une classe implémente des interfaces spécifiques, le conteneur sait que la classe doit être initialisée d'une certaine manière.

Malheureusement, l'automatisation n'est pas prévue pour tout, en particulier pour les paquets tiers. Le listener d'entité que nous venons d'écrire en est un exemple ; il ne peut pas être géré automatiquement par le conteneur de service Symfony car il n'implémente aucune interface et n'étend pas une "classe connue".

Nous devons déclarer partiellement le listener dans le conteneur. L'injection des dépendances peut être omise car elle peut être devinée par le conteneur, mais nous avons besoin d'ajouter manuellement quelques tags pour lier le listener avec le dispatcher d'événements Doctrine :

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -29,3 +29,7 @@ services:

         # add more service definitions when explicit configuration is needed
         # please note that last definitions always *replace* previous ones
    +    App\EntityListener\ConferenceEntityListener:
    +        tags:
    +            - { name: 'doctrine.orm.entity_listener', event: 'prePersist', entity: 'App\Entity\Conference'}
    +            - { name: 'doctrine.orm.entity_listener', event: 'preUpdate', entity: 'App\Entity\Conference'}

.. note::

    Ne confondez pas les listeners d'événements Doctrine et ceux de Symfony. Même s'ils se ressemblent beaucoup, ils n'utilisent pas la même infrastructure en interne.

Utiliser des slugs dans l'application
-------------------------------------

Essayez d'ajouter d'autres conférences dans l'interface d'administration et changez la ville ou l'année d'une conférence existante ; le slug ne sera pas mis à jour sauf si vous utilisez la valeur spéciale ``-``.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Annotations;@Route

La dernière modification consiste à mettre à jour les contrôleurs et les modèles pour utiliser le ``slug`` de la conférence pour les routes, au lieu de son ``id`` :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -31,7 +31,7 @@ class ConferenceController extends AbstractController
         }

         /**
    -     * @Route("/conference/{id}", name="conference")
    +     * @Route("/conference/{slug}", name="conference")
          */
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -10,7 +10,7 @@
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

L'accès à la page d'une conférence devrait maintenant se faire grâce à son slug :

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: Aller plus loin

    * Le `système d'événements Doctrine <https://symfony.com/doc/current/doctrine/events.html>`_ (*lifecycle callbacks* et *listeners*, *entity listeners* et *lifecycle subscribers*) ;

    * The `String component docs <https://symfony.com/doc/current/components/string.html>`_;

    * Le `conteneur de services <https://symfony.com/doc/current/service_container.html>`_ ;

    * La `cheat sheet des services de Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf>`_.
