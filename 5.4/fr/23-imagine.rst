Redimensionner des images
=========================

Dans le rendu de la page d'une conférence, les photos sont limitées à une taille maximale de 200 x 150 pixels. Ne faudrait-il pas optimiser les images, et réduire leur taille, si l'image originale est plus grande que celle qui est affichée ?

C'est une tâche idéale pour être ajoutée au workflow des commentaires, probablement juste après la validation du commentaire, et juste avant sa publication.

Ajoutons un nouvel état ``ready`` et une transition ``optimize`` :

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/workflow.yaml
    +++ b/config/packages/workflow.yaml
    @@ -16,6 +16,7 @@ framework:
                     - potential_spam
                     - spam
                     - rejected
    +                - ready
                     - published
                 transitions:
                     accept:
    @@ -29,13 +30,16 @@ framework:
                         to:   spam
                     publish:
                         from: potential_spam
    -                    to:   published
    +                    to:   ready
                     reject:
                         from: potential_spam
                         to:   rejected
                     publish_ham:
                         from: ham
    -                    to:   published
    +                    to:   ready
                     reject_ham:
                         from: ham
                         to:   rejected
    +                optimize:
    +                    from: ready
    +                    to:   published

.. index::
    single: Command;workflow:dump

Générez une représentation visuelle de la nouvelle configuration du workflow pour valider qu'elle décrit ce que nous voulons :

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Optimiser les images avec Imagine
---------------------------------

.. index::
    single: Imagine

L'optimisation des images se fera grâce à `GD`_ (vérifiez que l'extension GD est activée dans votre installation locale de PHP) et `Imagine`_ :

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.2"

Le redimensionnement d'une image peut être effectué via la classe de service suivante :

.. code-block:: php
    :caption: src/ImageOptimizer.php

    namespace App;

    use Imagine\Gd\Imagine;
    use Imagine\Image\Box;

    class ImageOptimizer
    {
        private const MAX_WIDTH = 200;
        private const MAX_HEIGHT = 150;

        private $imagine;

        public function __construct()
        {
            $this->imagine = new Imagine();
        }

        public function resize(string $filename): void
        {
            list($iwidth, $iheight) = getimagesize($filename);
            $ratio = $iwidth / $iheight;
            $width = self::MAX_WIDTH;
            $height = self::MAX_HEIGHT;
            if ($width / $height > $ratio) {
                $width = $height * $ratio;
            } else {
                $height = $width / $ratio;
            }

            $photo = $this->imagine->open($filename);
            $photo->resize(new Box($width, $height))->save($filename);
        }
    }

Après avoir optimisé la photo, nous stockons le nouveau fichier à la place de l'original. Par contre, vous voudrez peut-être garder l'image originale.

Ajouter une nouvelle étape au workflow
---------------------------------------

Modifiez le workflow pour gérer le nouvel état :

.. code-block:: diff
    :caption: patch_file

    --- a/src/MessageHandler/CommentMessageHandler.php
    +++ b/src/MessageHandler/CommentMessageHandler.php
    @@ -2,6 +2,7 @@

     namespace App\MessageHandler;

    +use App\ImageOptimizer;
     use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
    @@ -21,10 +22,12 @@ class CommentMessageHandler implements MessageHandlerInterface
         private $bus;
         private $workflow;
         private $mailer;
    +    private $imageOptimizer;
         private $adminEmail;
    +    private $photoDir;
         private $logger;

    -    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, string $adminEmail, LoggerInterface $logger = null)
    +    public function __construct(EntityManagerInterface $entityManager, SpamChecker $spamChecker, CommentRepository $commentRepository, MessageBusInterface $bus, WorkflowInterface $commentStateMachine, MailerInterface $mailer, ImageOptimizer $imageOptimizer, string $adminEmail, string $photoDir, LoggerInterface $logger = null)
         {
             $this->entityManager = $entityManager;
             $this->spamChecker = $spamChecker;
    @@ -32,7 +35,9 @@ class CommentMessageHandler implements MessageHandlerInterface
             $this->bus = $bus;
             $this->workflow = $commentStateMachine;
             $this->mailer = $mailer;
    +        $this->imageOptimizer = $imageOptimizer;
             $this->adminEmail = $adminEmail;
    +        $this->photoDir = $photoDir;
             $this->logger = $logger;
         }

    @@ -64,6 +69,12 @@ class CommentMessageHandler implements MessageHandlerInterface
                     ->to($this->adminEmail)
                     ->context(['comment' => $comment])
                 );
    +        } elseif ($this->workflow->can($comment, 'optimize')) {
    +            if ($comment->getPhotoFilename()) {
    +                $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());
    +            }
    +            $this->workflow->apply($comment, 'optimize');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Notez que ``$photoDir`` est automatiquement injecté parce que nous avons défini une *liaison* (bind) de conteneur sur ce nom de variable lors d'une étape précédente :

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    services:
        _defaults:
            bind:
                string $photoDir: "%kernel.project_dir%/public/uploads/photos"

Enregistrer des données uploadées en production
-------------------------------------------------

.. index::
    single: Platform.sh;File Service

Nous avons déjà défini un dossier en lecture-écriture dédié pour les fichiers uploadés dans ``.platform.app.yaml``. Mais le montage étant local, nous devons créer un *service de fichiers*, afin que le conteneur web et le *message consumer worker* puissent accéder au même support :

.. code-block:: diff
    :caption: patch_file

    --- a/.platform/services.yaml
    +++ b/.platform/services.yaml
    @@ -11,3 +11,7 @@ varnish:
             vcl: !include
                 type: string
                 path: config.vcl
    +
    +files:
    +    type: network-storage:1.0
    +    disk: 256

Utilisez-le pour le dossier d'upload des photos :

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -35,7 +35,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    -    "/public/uploads": { source: local, source_path: uploads }
    +    "/public/uploads": { source: service, service: files, source_path: uploads }

     relationships:
         database: "database:postgresql"

Cela devrait suffire pour faire fonctionner la fonctionnalité en production.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
