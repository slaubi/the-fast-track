Zmienianie rozmiaru obrazów
============================

Na stronie konferencji zdjęcia są ograniczone do maksymalnego rozmiaru 200 × 150 px. A co z optymalizacją obrazów i pomniejszaniem ich, jeśli przesłany oryginał jest większy niż ustalony limit?

Jest to doskonałe zadanie, które można dodać do przepływu pracy (ang. workflow) nad komentarzami, prawdopodobnie zaraz po zatwierdzeniu komentarza i tuż przed jego opublikowaniem.

Dodajmy zatem nowy stan: ``ready`` oraz przejście: ``optimize``:

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

Wygeneruj wizualną reprezentację nowego przepływu pracy, aby sprawdzić, czy opisuje ona to, czego chcemy:

.. code-block:: bash
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

Optymalizowanie obrazów z wykorzystaniem pakietu Imagine
---------------------------------------------------------

.. index::
    single: Imagine

Optymalizacja obrazów zostanie wykonana przy pomocy rozszerzenia `GD`_ (sprawdź, czy lokalna instalacja PHP posiada włączone to rozszerzenie) oraz pakietu `Imagine`_:

.. code-block:: bash

    $ symfony composer req "imagine/imagine:^1.2"

Zmiana rozmiaru obrazu może być wykonana za pośrednictwem następującego serwisu:

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

Po zoptymalizowaniu zdjęcia, nowy plik zapisujemy jako oryginalny, jednak jeśli chcesz, możesz zachować również wersję sprzed optymalizacji.

Dodawanie nowego kroku (ang. step) w przepływie pracy (ang. workflow)
----------------------------------------------------------------------

Zmodyfikuj przepływ pracy tak, aby obsługiwał nowy stan:

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

Zauważ, że zmienna ``$photoDir`` jest automatycznie wstrzykiwana, ponieważ zdefiniowaliśmy *powiązanie* (ang. bind) kontenera z tą zmienną w poprzednim kroku:

.. code-block:: yaml
    :caption: config/packages/services.yaml
    :class: ignore

    services:
        _defaults:
            bind:
                $photoDir: "%kernel.project_dir%/public/uploads/photos"

Przechowywanie przesłanych danych w środowisku produkcyjnym
-------------------------------------------------------------

.. index::
    single: SymfonyCloud;File Service

W pliku ``.symfony.cloud.yaml`` domyślnie mamy już skonfigurowany katalog dla przesłanych plików, jednak katalog ten jest zainstalowany (ang. mount) lokalnie. Jeśli chcemy, aby kontener i robotnik (ang. worker) odbierający polecenia mieli dostęp do tego samego katalogu, musimy utworzyć *serwis obsługi plików*:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -19,3 +19,7 @@ varnish:
             vcl: !include
                 type: string
                 path: config.vcl
    +
    +files:
    +    type: network-storage:1.0
    +    disk: 256

Użyj go jako katalogu dla przesłanych zdjęć:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -41,7 +41,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    -    "/public/uploads": { source: local, source_path: uploads }
    +    "/public/uploads": { source: service, service: files, source_path: uploads }

     hooks:
         build: |

To powinno wystarczyć, aby funkcja zadziałała w środowisku produkcyjnym.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
