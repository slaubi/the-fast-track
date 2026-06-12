Planifier des tâches
=====================

.. index::
    single: Cron

Certaines tâches de maintenance doivent s'exécuter selon un planning. Contrairement aux *workers*, qui tournent en continu, les tâches planifiées s'exécutent périodiquement pendant une courte durée.

Nettoyer les commentaires
-------------------------

Les commentaires marqués comme spam ou refusés par l'admin sont conservés dans la base de données, car l'admin peut vouloir les inspecter pendant un certain temps. Mais ils devraient probablement être supprimés au bout d'un moment. Les garder pendant une semaine après leur création devrait être suffisant.

Créez des méthodes utilitaires dans le repository des commentaires pour trouver les commentaires rejetés, les compter et les supprimer :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -5,7 +5,9 @@ namespace App\Repository;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    +use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -13,6 +15,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const COMMENTS_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -20,6 +24,27 @@ class CommentRepository extends ServiceEntityRepository
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
    +            ->setParameter('state_rejected', 'rejected')
    +            ->setParameter('state_spam', 'spam')
    +            ->setParameter('date', new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'))
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
    use Symfony\Component\Console\Attribute\Option;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand
    {
        public function __invoke(
            SymfonyStyle $io,
            CommentRepository $commentRepository,
            #[Option(description: 'Dry run')]
            bool $dryRun = false,
        ): int {
            if ($dryRun) {
                $io->note('Dry mode enabled');

                $count = $commentRepository->countOldRejected();
            } else {
                $count = $commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

Toutes les commandes de l'application sont enregistrées avec les commandes par défaut de Symfony, et sont toutes accessibles avec ``symfony console``. Comme le nombre de commandes disponibles peut être important, vous devez les mettre dans le bon *namespace*. Par convention, les commandes spécifiques à l'application devraient être stockées sous le namespace ``app``. Ajoutez autant de sous-namespaces que vous le souhaitez en les séparant par deux points (``:``).

Une commande déclare ses *arguments* et ses *options* avec les attributs ``#[Argument]`` et ``#[Option]`` sur les paramètres de ``__invoke()`` (le paramètre ``$dryRun`` devient l'option ``--dry-run``). Symfony injecte les autres paramètres en fonction de leur type : ``SymfonyStyle`` pour écrire une sortie joliment formatée dans la console, et n'importe quel service, comme le repository des commentaires, de la même manière que pour les arguments des contrôleurs.

Nettoyez la base de données en exécutant la commande :

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Planifier la commande
---------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Exécuter la commande à la main fonctionne, mais elle devrait s'exécuter chaque nuit. Le composant Symfony Scheduler génère des messages selon un planning ; ils sont ensuite consommés par un worker, comme n'importe quel autre message Messenger.

Ajoutez le composant Scheduler, ainsi que la bibliothèque qui analyse les expressions cron :

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Planifiez la commande avec l'attribut ``#[AsCronTask]`` :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/CommentCleanupCommand.php
    +++ w/src/Command/CommentCleanupCommand.php
    @@ -7,8 +7,10 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Attribute\Option;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Style\SymfonyStyle;
    +use Symfony\Component\Scheduler\Attribute\AsCronTask;

     #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    +#[AsCronTask('50 23 * * *')]
     class CommentCleanupCommand
     {
         public function __invoke(

L'attribut enregistre la commande sur le *planning* (« schedule ») par défaut avec une expression cron : chaque nuit à 23 h 50 (UTC). Vérifiez-le :

.. code-block:: terminal

    $ symfony console debug:scheduler

Un planning est exposé comme un transport Messenger ordinaire qui porte son nom ; consommez-le comme n'importe quel autre transport :

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Déployer le planning
--------------------

.. index::
    single: Upsun;Workers

Sur Upsun, le worker ne consomme que le transport ``async``. Faites-lui consommer aussi le planning :

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -87,4 +87,4 @@ applications:
             messenger:
                 commands:
                     # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
    -                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async
    +                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async scheduler_default

C'est tout ce qu'il faut : pas de crontab, pas de processus supplémentaire ; le planning vit dans le code PHP, à côté de la tâche qu'il déclenche, et il est déployé et versionné comme le reste de l'application.

Et les crons système ?
----------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun prend aussi en charge les *cron jobs* au niveau du système d'exploitation, décrits dans ``.upsun/config.yaml`` aux côtés du conteneur web et des workers ; la configuration par défaut en définit déjà un qui nettoie les sessions PHP expirées. Les crons système conviennent bien aux tâches qui ne sont pas implémentées en PHP.

L'utilitaire ``croncape`` utilisé par le cron par défaut surveille l'exécution de la commande et envoie un email aux adresses définies dans la variable d'environnement ``MAILTO`` si la commande retourne un code de sortie différent de ``0`` :

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Notez que les crons sont installés sur toutes les branches d'Upsun. Si vous ne voulez pas en exécuter sur des environnements hors production, vérifiez la variable d'environnement ``$PLATFORM_ENVIRONMENT_TYPE`` :

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Aller plus loin

    * La `documentation du composant Scheduler`_ ;

    * `Syntaxe cron/crontab`_ ;

    * `Dépôt de croncape`_ ;

    * `Commandes de la console Symfony`_ ;

    * La `cheat sheet de la console Symfony`_.

.. _`documentation du composant Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`Syntaxe cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`Dépôt de croncape`: https://github.com/symfonycorp/croncape
.. _`Commandes de la console Symfony`: https://symfony.com/doc/current/console.html
.. _`cheat sheet de la console Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
