Écouter les événements
=========================

Il manque une barre de navigation au layout actuel pour revenir à la page d'accueil ou pour passer d'une conférence à l'autre.

Ajouter un en-tête au site web
-------------------------------

.. index::
    single: Twig;for
    single: Twig;path

Tout ce qui doit être affiché sur toutes les pages web, comme un en-tête, doit faire partie du layout de base principal :

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -12,6 +12,15 @@
             {% endblock %}
         </head>
         <body>
    +        <header>
    +            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    +            <ul>
    +            {% for conference in conferences %}
    +                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +            {% endfor %}
    +            </ul>
    +            <hr />
    +        </header>
             {% block body %}{% endblock %}
         </body>
     </html>

L'ajout de ce code au layout signifie que tous les templates qui l'étendent doivent définir une variable ``conferences``, créée et  transmise par leurs contrôleurs.

Comme nous n'avons que deux contrôleurs, vous *pourriez* procéder comme ceci (ne modifiez pas votre code car nous verrons très vite une meilleure façon de faire) :

.. code-block:: diff
    :class: ignore

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -21,12 +21,13 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository, #[MapQueryParameter] int $offset = 0): Response
         {
             $offset = max(0, $offset);
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return $this->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,

Imaginez devoir mettre à jour des dizaines de contrôleurs. Et faire la même chose sur tous les nouveaux. Ce n'est pas très pratique. Il doit y avoir un meilleur moyen.

Twig a la notion de variables globales. Une *variable globale* est disponible dans tous les templates générés. Vous pouvez les définir dans un fichier de configuration, mais cela ne fonctionne que pour les valeurs statiques. Pour ajouter toutes les conférences comme variable globale Twig, nous allons créer un *listener*.

Découvrir les événements Symfony
-----------------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony intègre un composant Event Dispatcher. Un *dispatcher* répartit certains *événements* à des moments précis que les *listeners* peuvent écouter. Les *listeners* sont des *hooks* dans le cœur du framework.

Par exemple, certains événements vous permettent d'interagir avec le cycle de vie des requêtes HTTP. Pendant le traitement d'une requête, le dispatcher répartit les événements lorsqu'une requête a été créée, lorsqu'un contrôleur est sur le point d'être exécuté, lorsqu'une réponse est prête à être envoyée, ou lorsqu'une exception a été levée. Un listener peut écouter un ou plusieurs événements et exécuter une logique basée sur le contexte de l'événement.

Les événements sont des points d'extension bien définis qui rendent le framework plus générique et extensible. De nombreux composants Symfony tels que Security, Messenger, Workflow ou Mailer les utilisent largement.

Un autre exemple intégré d'événements et de listeners en action est le cycle de vie d'une commande : vous pouvez créer un listener pour exécuter du code avant *n'importe quelle* commande.

Tout paquet ou bundle peut également déclencher ses propres événements pour rendre son code extensible.

Pour éviter d'avoir un fichier de configuration qui décrit les événements qu'un listener veut écouter, ajoutez l'attribut ``#[AsEventListener]`` sur la classe ou la méthode du listener. Ceci permet aux listeners d'être enregistrés automatiquement dans le dispatcher Symfony.

Implémenter un listener
-----------------------

.. index::
    single: Event;Listener
    single: Listener
    single: Command;make:listener

Vous connaissez la chanson par cœur maintenant, utilisez le *Maker Bundle* pour générer un listener :

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:listener TwigEventListener

La commande vous demande quel événement vous voulez écouter. Choisissez l'événement ``Symfony\Component\HttpKernel\Event\ControllerEvent`` qui est envoyé juste avant l'appel d'un contrôleur. C'est le meilleur moment pour injecter la variable globale ``conferences`` afin que Twig y ait accès lorsque le contrôleur générera le template. Mettez votre listener à jour comme suit :

.. code-block:: diff
    :caption: patch_file

    --- i/src/EventListener/TwigEventListener.php
    +++ w/src/EventListener/TwigEventListener.php
    @@ -2,14 +2,22 @@

     namespace App\EventListener;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     final class TwigEventListener
     {
    +    public function __construct(
    +        private Environment $twig,
    +        private ConferenceRepository $conferenceRepository,
    +    ) {
    +    }
    +
         #[AsEventListener]
         public function onControllerEvent(ControllerEvent $event): void
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }
     }

Maintenant, vous pouvez ajouter autant de contrôleurs que vous le souhaitez : la variable ``conferences`` sera toujours disponible dans Twig.

.. note::

    Nous parlerons d'une alternative bien plus performante dans une prochaine étape.

Trier les conférences par année et par ville
----------------------------------------------

Le tri de la liste des conférences par année peut faciliter la navigation. Nous pourrions créer notre propre méthode pour récupérer et trier toutes les conférences, mais nous allons plutôt remplacer l'implémentation par défaut de la méthode ``findAll()``, afin que le tri s'applique partout :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/ConferenceRepository.php
    +++ w/src/Repository/ConferenceRepository.php
    @@ -16,6 +16,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll(): array
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         //    /**
         //     * @return Conference[] Returns an array of Conference objects
         //     */

À la fin de cette étape, le site web devrait ressembler à ceci :

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: Aller plus loin

    * Le `flux Request-Response`_ dans les applications Symfony ;

    * Les `événements HTTP intégrés à Symfony`_ ;

    * Les `événements de la console intégrés à Symfony`_.

.. _`flux Request-Response`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`événements HTTP intégrés à Symfony`: https://symfony.com/doc/current/reference/events.html
.. _`événements de la console intégrés à Symfony`: https://symfony.com/doc/current/components/console/events.html
