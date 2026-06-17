Ejecutando trabajos Cron
========================

.. index::
    single: Cron

Los trabajos Cron son útiles para realizar tareas de mantenimiento. A diferencia de los *workers*, se ejecutan en un horario específico durante un corto período de tiempo.

Limpiando comentarios
---------------------

Los comentarios marcados como spam o rechazados por el administrador se mantienen en la base de datos, ya que el administrador puede querer inspeccionarlos durante un tiempo. Pero probablemente deberían ser eliminados después de algún tiempo. Mantenerlos durante una semana después de su creación es probablemente suficiente.

Crea algunos métodos útiles en el repositorio de comentarios para encontrar los comentarios rechazados, contarlos y eliminarlos:

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
    @@ -16,6 +17,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const PAGINATOR_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -23,6 +26,29 @@ class CommentRepository extends ServiceEntityRepository
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
    +                'date' => new \DateTime(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'),
    +            ])
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    Para consultas más complejas, a veces es útil echar un vistazo a las sentencias SQL generadas (se pueden encontrar en los registros y en el perfilador de solicitudes Web).

Usando constantes de clase, parámetros de contenedor y variables de entorno
----------------------------------------------------------------------------

.. index::
    single: Container;Parameters

¿7 días? Podríamos haber elegido otro número, tal vez 10 o 20. Este número podría evolucionar con el tiempo. Hemos decidido almacenarlo como una constante en la clase, pero podríamos haberlo almacenado como un parámetro en el contenedor, o incluso podríamos haberlo definido como una variable de entorno.

Aquí hay algunas reglas generales para decidir qué abstracción usar:

* Si el valor es sensible (contraseñas, tokens de API...), utiliza el *almacenamiento secreto* de Symfony o un *Vault*;

* Si el valor es dinámico y deberías poder cambiarlo *sin necesidad* de volver a desplegar, utiliza una *variable de entorno*;

* Si el valor puede ser diferente entre entornos, utiliza un *parámetro contenedor*;

* Para todo lo demás, almacena el valor en el código, como una *constante de clase*.

Creando un comando de línea de comandos
----------------------------------------

Eliminar los comentarios antiguos es la tarea perfecta para una trabajo cron. Debería hacerse de forma regular, y un pequeño retraso no tiene un impacto importante.

Crea un comando de línea de comandos llamado ``app:comment:cleanup`` generando para ello el archivo ``src/Command/CommentCleanupCommand.php``:

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;

    class CommentCleanupCommand extends Command
    {
        private $commentRepository;

        protected static $defaultName = 'app:comment:cleanup';

        public function __construct(CommentRepository $commentRepository)
        {
            $this->commentRepository = $commentRepository;

            parent::__construct();
        }

        protected function configure()
        {
            $this
                ->setDescription('Deletes rejected and spam comments from the database')
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

            return 0;
        }
    }

Todos los comandos de la aplicación están registrados junto con los comandos incorporados de Symfony y todos ellos son accesibles a través de ``symfony console``. Como el número de comandos disponibles puede ser grande, debes crear un espacio de nombres para ellos. Por convención, los comandos de la aplicación deben almacenarse bajo el espacio de nombres ``app``. Añade cualquier número de subespacios de nombres separándolos con dos puntos ( ``:`` ).

Un comando obtiene la *entrada* (argumentos y opciones pasadas al comando) y puede usar la *salida* para escribir en la consola.

Limpia la base de datos ejecutando el comando:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Configurando un Cron en SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

Una de las ventajas de SymfonyCloud es que la mayor parte de la configuración se almacena en un solo archivo: ``.symfony.cloud.yaml``. El contenedor web, los *workers* y los trabajos cron se describen juntos para ayudar a su mantenimiento:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -52,6 +52,15 @@ hooks:

             (>&2 symfony-deploy)

    +crons:
    +    comment_cleanup:
    +        # Cleanup every night at 11.50 pm (UTC).
    +        spec: '50 23 * * *'
    +        cmd: |
    +            if [ "$SYMFONY_BRANCH" = "master" ]; then
    +                croncape symfony console app:comment:cleanup
    +            fi
    +
     workers:
         messages:
             commands:

La sección de ``crons`` define todos los trabajos del cron. Cada cron se ejecuta según un calendario ``spec``.

La utilidad ``croncape`` monitoriza la ejecución del comando y envía un correo electrónico a las direcciones definidas en la variable de entorno ``MAILTO`` si el comando devuelve un código de salida distinto de ``0``.

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

Configura la variable de entorno ``MAILTO``:

.. code-block:: terminal

    $ symfony var:set MAILTO=ops@example.com

Puedes forzar la ejecución de un cron desde tu máquina local:

.. code-block:: terminal
    :class: ignore

    $ symfony cron comment_cleanup

Ten en cuenta que los crons están configurados en todas las ramas de SymfonyCloud. Si no deseas ejecutar algunos en entornos que no sean de producción, comprueba la variable de entorno ``$SYMFONY_BRANCH``:

.. code-block:: terminal
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Yendo más allá

    * `Sintaxis de Cron/crontab <https://en.wikipedia.org/wiki/Cron>`_;

    * `Repositorio Croncape <https://github.com/symfonycorp/croncape>`_ ;

    * `Comandos de Symfony Console <https://symfony.com/doc/current/console.html>`_;

    * La `Chuleta de Symfony Console <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_.
