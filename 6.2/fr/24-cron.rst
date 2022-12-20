Exécuter des crons
===================

.. index::
    single: Cron

Les crons sont utiles pour les tâches de maintenance. Contrairement aux *workers*, ils travaillent selon un horaire établi pour une courte période de temps.

Nettoyer les commentaires
-------------------------

Les commentaires marqués comme spam ou refusés par l'admin sont conservés dans la base de données, car l'admin peut vouloir les inspecter pendant un certain temps. Mais ils devraient probablement être supprimés au bout d'un moment. Les garder pendant une semaine après leur création devrait être suffisant.

Créez des méthodes utilitaires dans le repository des commentaires pour trouver les commentaires rejetés, les compter et les supprimer :

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -6,6 +6,7 @@ use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -18,6 +19,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -25,6 +28,29 @@ class CommentRepository extends ServiceEntityRepository
             parent::__construct($registry, Comment::class);
         }

    +    public function countOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->select('COUNT(c.id)')->getQuery()->getSingleScalarResult();
    +    }
    +
    +    public function deleteOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->delete()->getQuery()->execute();
    +    }
    +
    +    private function getOldRejectedQueryBuilder(): QueryBuilder
    +    {
    +        return $this->createQueryBuilder('c')
    +            ->andWhere('c.state = :state_rejected or c.state = :state_spam')
    +            ->andWhere('c.createdAt < :date')
    +            ->setParameters([
    +                'state_rejected' => 'rejected',
    +                'state_spam' => 'spam',
    +                'date' => new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    Pour les requêtes plus complexes, il est parfois utile de jeter un coup d'œil aux requêtes SQL générées (elles se trouvent dans les logs et dans le profileur de requêtes web).

Utiliser des constantes de classe, des paramètres de conteneur et des variables d'environnement
------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 jours ? Nous aurions pu choisir un autre chiffre, pourquoi pas 10 ou 20 ? Ce nombre pourrait évoluer avec le temps. Nous avons décidé de le stocker en tant que constante dans la classe, mais nous aurions peut-être pu le stocker en tant que paramètre dans le conteneur, ou même le définir en tant que variable d'environnement.

Voici quelques règles de base pour décider quelle abstraction utiliser :

* Si la valeur est sensible (mots de passe, jetons API, etc.), utilisez le *stockage de chaîne secrète* de Symfony ou un Vault ;

* Si la valeur est dynamique et que vous devriez pouvoir la modifier *sans* redéployer, utilisez une *variable d'environnement* ;

* Si la valeur peut être différente d'un environnement à l'autre, utilisez un *paramètre de conteneur* ;

* Pour tout le reste, stockez la valeur dans le code, comme dans une *constante de classe*.

Créer une commande de console
------------------------------

Supprimer les anciens commentaires est une tâche idéale pour un *cron job*. Il faut le faire de façon régulière, et un petit retard n'a pas d'impact majeur.

Créez une commande nommée ``app:comment:cleanup`` en créant un fichier ``src/Command/CommentCleanupCommand.php`` :

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand extends Command
    {
        public function __construct(
            private CommentRepository $commentRepository,
        ) {
            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->addOption('dry-run', null, InputOption::VALUE_NONE, 'Dry run')
            ;
        }

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $io = new SymfonyStyle($input, $output);

            if ($input->getOption('dry-run')) {
                $io->note('Dry mode enabled');

                $count = $this->commentRepository->countOldRejected();
            } else {
                $count = $this->commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Toutes les commandes de l'application sont enregistrées avec les commandes par défaut de Symfony, et sont toutes accessibles avec ``symfony console``. Comme le nombre de commandes disponibles peut être important, vous devez les mettre dans le bon *namespace*. Par convention, les commandes spécifiques à l'application devraient être stockées sous le namespace ``app``. Ajoutez autant de sous-namespaces que vous le souhaitez en les séparant par deux points (``:``).

Une commande reçoit l'*entrée* (les arguments et les options passés à la commande) et vous pouvez utiliser la *sortie* pour écrire dans la console.

Nettoyez la base de données en exécutant la commande :

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Configurer un cron sur Platform.sh
----------------------------------

.. index::
    single: Platform.sh;Cron
    single: Platform.sh;Croncape

L'un des avantages de Platform.sh est qu'une bonne partie de la configuration est stockée dans un seul fichier : ``.platform.app.yaml``. Le conteneur web, les *workers* et les *cron jobs* sont décrits au même endroit pour faciliter la maintenance :

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -61,6 +61,14 @@ crons:
             spec: '50 23 * * *'
             cmd: if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then croncape php-security-checker; fi

    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messenger:
             commands:

La section ``crons`` définit tous les *cron jobs*. Chaque cron fonctionne selon un planning spécifique (``spec``).

L'utilitaire ``croncape`` surveille l'exécution de la commande et envoie un email aux adresses définies dans la variable d'environnement ``MAILTO`` si la commande retourne un code de sortie différent de ``0``.

.. index::
    single: Symfony CLI;cloud:variable:create
    single: Symfony CLI;cron

Configurez la variable d'environnement ``MAILTO`` :

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Notez que les crons sont installés sur toutes les branches de Platform.sh. Si vous ne voulez pas en exécuter sur des environnements hors production, vérifiez la variable d'environnement ``$PLATFORM_ENVIRONMENT_TYPE`` :

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Aller plus loin

    * `Syntaxe cron/crontab`_ ;

    * `Dépôt de croncape`_ ;

    * `Commandes de la console Symfony`_ ;

    * La `cheat sheet de la console Symfony`_.

.. _`Syntaxe cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Dépôt de croncape`: https://github.com/symfonycorp/croncape
.. _`Commandes de la console Symfony`: https://symfony.com/doc/current/console.html
.. _`cheat sheet de la console Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
