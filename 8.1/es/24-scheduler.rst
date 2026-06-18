Programando tareas
==================

.. index::
    single: Cron

Algunas tareas de mantenimiento deben ejecutarse según un horario. A diferencia de los *workers*, que se ejecutan continuamente, las tareas programadas se ejecutan periódicamente durante un corto período de tiempo.

Limpiando comentarios
---------------------

Los comentarios marcados como spam o rechazados por el administrador se mantienen en la base de datos, ya que el administrador puede querer inspeccionarlos durante un tiempo. Pero probablemente deberían ser eliminados después de algún tiempo. Mantenerlos durante una semana después de su creación es probablemente suficiente.

Crea algunos métodos útiles en el repositorio de comentarios para encontrar los comentarios rechazados, contarlos y eliminarlos:

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

    Para consultas más complejas, a veces es útil echar un vistazo a las sentencias SQL generadas (se pueden encontrar en los registros y en el perfilador de solicitudes Web).

Usando constantes de clase, parámetros de contenedor y variables de entorno
----------------------------------------------------------------------------

.. index::
    single: Container;Parameters

¿7 días? Podríamos haber elegido otro número, tal vez 10 o 20. Este número podría evolucionar con el tiempo. Hemos decidido almacenarlo como una constante en la clase, pero podríamos haberlo almacenado como un parámetro en el contenedor, o incluso podríamos haberlo definido como una variable de entorno.

Aquí tienes algunas reglas generales para decidir qué abstracción utilizar:

* Si el valor es sensible (contraseñas, tokens de API...), utiliza el *almacenamiento secreto* de Symfony o un *Vault*;

* Si el valor es dinámico y deberías poder cambiarlo *sin necesidad* de volver a desplegar, utiliza una *variable de entorno*;

* Si el valor puede ser diferente entre entornos, utiliza un *parámetro de contenedor*;

* Para todo lo demás, almacena el valor en el código, como una *constante de clase*.

Creando un comando de línea de comandos
----------------------------------------

Eliminar los comentarios antiguos es la tarea perfecta para un trabajo cron. Debería hacerse de forma regular, y un pequeño retraso no tiene un impacto importante.

Crea un comando de línea de comandos llamado ``app:comment:cleanup`` generando para ello el archivo ``src/Command/CommentCleanupCommand.php``:

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

Todos los comandos de la aplicación están registrados junto con los comandos incorporados de Symfony y todos ellos son accesibles a través de ``symfony console``. Como el número de comandos disponibles puede ser grande, debes crear un espacio de nombres para ellos. Por convención, los comandos de la aplicación deben almacenarse bajo el espacio de nombres ``app``. Añade cualquier número de subespacios de nombres separándolos con dos puntos ( ``:`` ).

Un comando declara sus *argumentos* y *opciones* con los atributos ``#[Argument]`` y ``#[Option]`` sobre los parámetros de ``__invoke()`` (el parámetro ``$dryRun`` se convierte en la opción ``--dry-run``). Symfony inyecta el resto de los parámetros en función de su tipo: ``SymfonyStyle`` para escribir una salida con buen formato en la consola, y cualquier servicio, como el repositorio de comentarios, de la misma manera que lo hace con los argumentos del controlador.

Limpia la base de datos ejecutando el comando:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

Programando el comando
----------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

Ejecutar el comando a mano funciona, pero debería ejecutarse cada noche. El componente Symfony Scheduler genera mensajes según un horario; luego son consumidos por un *worker*, como cualquier otro mensaje de Messenger.

Añade el componente Scheduler, junto con la librería que analiza las expresiones cron:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

Programa el comando con el atributo ``#[AsCronTask]``:

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

El atributo registra el comando en el *schedule* (planificación) predeterminado con una expresión cron: cada noche a las 11.50 pm (UTC). Compruébalo:

.. code-block:: terminal

    $ symfony console debug:scheduler

Una planificación se expone como un transporte de Messenger normal que lleva su nombre; consúmelo como cualquier otro transporte:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

Desplegando la planificación
----------------------------

.. index::
    single: Upsun;Workers

En Upsun, el *worker* solo consume el transporte ``async``. Haz que consuma también la planificación:

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

Eso es todo lo que hace falta: sin crontab, sin proceso adicional; la planificación vive en el código PHP, junto a la tarea que dispara, y se despliega y versiona como el resto de la aplicación.

¿Y los crons del sistema?
-------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun también soporta trabajos cron a nivel del sistema operativo, descritos en ``.upsun/config.yaml`` junto al contenedor web y los *workers*; la configuración predeterminada ya define uno que limpia las sesiones PHP expiradas. Los crons del sistema son una buena opción para tareas que no están implementadas en PHP.

La utilidad ``croncape`` que usa el cron predeterminado monitoriza la ejecución del comando y envía un correo electrónico a las direcciones definidas en la variable de entorno ``MAILTO`` si el comando devuelve un código de salida distinto de ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

Ten en cuenta que los crons están configurados en todas las ramas de Upsun. Si no deseas ejecutar algunos en entornos que no sean de producción, comprueba la variable de entorno ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Yendo más allá

    * La `documentación del componente Scheduler`_ ;

    * La `sintaxis de Cron/crontab`_ ;

    * El `repositorio de Croncape`_ ;

    * Los `comandos de Symfony Console`_ ;

    * La `Chuleta de Symfony Console`_ .

.. _`documentación del componente Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`sintaxis de Cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`repositorio de Croncape`: https://github.com/symfonycorp/croncape
.. _`comandos de Symfony Console`: https://symfony.com/doc/current/console.html
.. _`Chuleta de Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
