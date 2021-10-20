Resizing Images 
=============== 
 
On the conference page design, photos are constrained to a maximum size of 200 by 150 pixels. What about optimizing the images and reducing their size if the uploaded original is larger than the limits? 
 
That is a perfect job that can be added to the comment workflow, probably just after the comment is validated and just before it is published. 
 
Let's add a new ``ready`` state and an ``optimize`` transition: 
 
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
 
Generate a visual representation of the new workflow configuration to validate that it describes what we want: 
 
.. code-block:: bash 
    :class: ignore 
 
    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png 
 
.. image:: images/workflow-final.png 
    :align: center 
 
Optimizing Images with Imagine 
------------------------------ 
 
.. index:: 
    single: Imagine 
 
Image optimizations will be done thanks to `GD`_ (check that your local PHP installation has the GD extension enabled) and `Imagine`_: 
 
.. code-block:: bash 
 
    $ symfony composer req "imagine/imagine:^1.2" 
 
Resizing an image can be done via the following service class: 
 
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
 
After optimizing the photo, we store the new file in place of the original one. You might want to keep the original image around though. 
 
Adding a new Step in the Workflow 
--------------------------------- 
 
Modify the workflow to handle the new state: 
 
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
 
Note that ``$photoDir`` is automatically injected as we defined a container *bind* on this variable name in a previous step: 
 
.. code-block:: yaml 
    :caption: config/packages/services.yaml 
    :class: ignore 
 
    services: 
        _defaults: 
            bind: 
                $photoDir: "%kernel.project_dir%/public/uploads/photos" 
 
Storing Uploaded Data in Production 
----------------------------------- 
 
.. index:: 
    single: SymfonyCloud;File Service 
 
We have already defined a special read-write directory for uploaded files in ``.symfony.cloud.yaml``. But the mount is local. If we want the web container and the message consumer worker to be able to access the same mount, we need to create a *file service*: 
 
.. code-block:: diff 
    :caption: patch_file 
 
    --- a/.symfony/services.yaml 
    +++ b/.symfony/services.yaml 
    @@ -11,3 +11,7 @@ varnish: 
             vcl: !include 
                 type: string 
                 path: config.vcl 
    + 
    +files: 
    +    type: network-storage:1.0 
    +    disk: 256 
 
Use it for the photos upload directory: 
 
.. code-block:: diff 
    :caption: patch_file 
 
    --- a/.symfony.cloud.yaml 
    +++ b/.symfony.cloud.yaml 
    @@ -37,7 +37,7 @@ web: 
 
     mounts: 
         "/var": { source: local, source_path: var } 
    -    "/public/uploads": { source: local, source_path: uploads } 
    +    "/public/uploads": { source: service, service: files, source_path: uploads } 
 
     hooks: 
         build: | 
 
This should be enough to make the feature work in production. 
 
.. _`GD`: https://libgd.github.io/ 
.. _`Imagine`: https://github.com/avalanche123/Imagine 
