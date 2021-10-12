Executando Crons
================

.. index::
    single: Cron

Crons são úteis para realizar tarefas de manutenção. Ao contrário dos workers, eles cumprem um cronograma por um curto período de tempo.

Limpando Comentários
---------------------

Comentários marcados como spam ou rejeitados pelo administrador são mantidos no banco de dados, pois o administrador pode querer inspecioná-los por algum tempo. Mas eles provavelmente devem ser removidos após algum tempo. Mantê-los durante uma semana após a sua criação provavelmente é o suficiente.

Crie alguns métodos utilitários no repositório de comentários para encontrar comentários rejeitados, contá-los e excluí-los:

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

    Para consultas mais complexas, às vezes é útil dar uma olhada nas instruções SQL geradas (elas podem ser encontradas nos logs e no profiler para requisições web).

Utilizando Constantes de Classe, Parâmetros do Container e Variáveis de Ambiente
----------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 dias? Nós poderíamos ter escolhido outro número, talvez 10 ou 20. Esse número pode evoluir com o tempo. Decidimos armazená-lo como uma constante na classe, mas poderíamos tê-lo armazenado como um parâmetro no container, ou até mesmo tê-lo definido como uma variável de ambiente.

Aqui estão algumas regras práticas para decidir qual abstração usar:

* Se o valor for sensível (senhas, tokens de API, ...), use o *armazenamento de segredos* do Symfony ou um cofre;

* Se o valor for dinâmico e você puder alterá-lo *sem* reimplantação, use uma *variável de ambiente*;

* Se o valor puder ser diferente entre ambientes, use um *parâmetro do container*;

* Para todo o resto, armazene o valor em código, por exemplo, em uma *constante de classe*.

Criando um Comando da CLI
-------------------------

Remover os comentários antigos é a tarefa perfeita para um cron job. Isso deve ser feito regularmente, e um pequeno atraso não tem qualquer impacto significativo.

Crie um comando da CLI chamado `app:comment:cleanup`` criando um arquivo ``src/Command/CommentCleanupCommand.php``:

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

Todos os comandos da aplicação são registrados juntamente com os comandos internos do Symfony, e todos são acessíveis via ``symfony console``. Como o número de comandos disponíveis pode ser grande, você deve usar namespaces. Por convenção, os comandos da aplicação devem ser armazenados sob o namespace ``app``. Adicione qualquer número de sub-namespaces separando-os por dois pontos (``:``).

Um comando recebe a *entrada* (argumentos e opções passados para o comando) e você pode usar a *saída* para escrever no console.

Limpe o banco de dados executando o comando:

.. code-block:: bash

    $ symfony console app:comment:cleanup

Configurando um Cron na SymfonyCloud
------------------------------------

.. index::
    single: SymfonyCloud;Cron
    single: SymfonyCloud;Croncape

Uma das coisas boas sobre a SymfonyCloud é que a maior parte da configuração é armazenada em um único arquivo: ``.symfony.cloud.yaml``. O container web, os workers, e os cron jobs são descritos juntos para ajudar na manutenção:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -56,6 +56,15 @@ hooks:

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

A seção ``crons`` define todos os cron jobs. Cada cron é executado de acordo com um agendamento ``spec``.

O utilitário ``croncape`` monitora a execução do comando e envia um e-mail para os endereços definidos na variável de ambiente ``MAILTO`` se o comando retornar qualquer código de saída diferente de ``0``.

.. index::
    single: Symfony CLI;var:set
    single: Symfony CLI;cron

Configure a variável de ambiente ``MAILTO``:

.. code-block:: bash

    $ symfony var:set MAILTO=ops@example.com

Você pode forçar um cron a executar a partir de sua máquina local:

.. code-block:: bash
    :class: ignore

    $ symfony cron comment_cleanup

Note que os crons são configurados em todas as branchs da SymfonyCloud. Se você não quiser executar alguns deles em ambientes que não são de produção, verifique a variável de ambiente ``$SYMFONY_BRANCH``:

.. code-block:: bash
    :class: ignore

    if [ "$SYMFONY_BRANCH" = "master" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: Indo Além

    * `Sintaxe Cron/crontab <https://en.wikipedia.org/wiki/Cron>`_;

    * `Repositório Croncape <https://github.com/symfonycorp/croncape>`_;

    * `Comandos do Console do Symfony <https://symfony.com/doc/current/console.html>`_;

    * A `Cheat Sheet sobre o Console do Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf>`_.
