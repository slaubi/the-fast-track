Utilizando Branches no Código
==============================

Existem muitas formas de organizar o fluxo de trabalho das mudanças no código em um projeto. Mas trabalhar diretamente na branch master do Git e implantar diretamente em produção sem testes provavelmente não é a melhor.

Testar não se trata apenas de testes unitários ou funcionais, mas também de verificar o comportamento da aplicação com dados de produção. Se você ou seus `stakeholders`_ puderem navegar pela aplicação exatamente como ela será implantada para os usuários finais, isso se tornará uma grande vantagem e permitirá que você implante com confiança. É especialmente poderoso quando pessoas não técnicas podem validar novas funcionalidades.

Continuaremos a fazer todo o trabalho na branch master do Git nos próximos passos, por uma questão de simplicidade e para evitar repetições, mas vamos ver como isso poderia funcionar melhor.

Adotando um Fluxo de Trabalho para o Git
----------------------------------------

Um possível fluxo de trabalho é criar uma branch para cada nova funcionalidade ou correção de bug. É simples e eficiente.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Criando Branches
----------------

.. index::
    single: Git;branch
    single: Git;checkout

O fluxo de trabalho começa com a criação de uma branch no Git:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Este comando cria uma branch ``sessions-in-redis`` a partir da branch ``master``. Ele faz um fork do código e da configuração da infraestrutura.

Armazenando Sessões no Redis
-----------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Como você pode ter adivinhado pelo nome da branch, nós queremos mudar o armazenamento de sessão do sistema de arquivos para um armazenamento do Redis.

Os passos necessários para tornar isso realidade são típicos:

#. Criar uma branch no Git;

#. Atualizar a configuração do Symfony se necessário;

#. Escrever e/ou atualizar algum código, se necessário;

#. Atualizar a configuração do PHP (adicionar a extensão PHP Redis);

#. Atualizar a infraestrutura no Docker e na SymfonyCloud (adicionar o serviço Redis);

#. Testar localmente;

#. Testar remotamente;

#. Fazer o merge da branch na master;

#. Implantar em produção;

#. Remover a branch.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Vou deixá-lo testar localmente navegando no site. Como não há mudanças visuais e como ainda não estamos usando sessões, tudo deve funcionar como antes.

Implantando uma Branch
----------------------

.. index::
    single: SymfonyCloud;Environment

Antes de implantarmos em produção, devemos testar a branch na mesma infraestrutura que a de produção. Devemos também validar que tudo funciona corretamente no ambiente ``prod`` do Symfony (o site local usou o ambiente ``dev`` do Symfony).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Agora, vamos criar um *ambiente SymfonyCloud* com base na *branch do Git*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Este comando cria um novo ambiente da seguinte forma:

* A branch herda o código e a infraestrutura da branch atual do Git (``sessions-in-redis``);

* Os dados provêm do ambiente master (também conhecido como produção), tirando um snapshot consistente de todos os dados de serviço, incluindo arquivos (que o usuário fez upload, por exemplo) e bancos de dados;

* Um novo cluster dedicado é criado para implantar o código, os dados e a infraestrutura.

Como a implantação segue as mesmas etapas da implantação para produção, as migrações do banco de dados também serão executadas. Essa é uma ótima forma de validar se as migrações funcionam com os dados de produção.

Os ambientes que não são ``master`` são muito semelhantes ao ``master``, exceto por algumas pequenas diferenças: por exemplo, os e-mails não são enviados por padrão.

.. index::
    single: Symfony CLI;open:remote

Após a conclusão da implantação, abra a nova branch em um navegador:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Note que todos os comandos da SymfonyCloud funcionam na branch atual do Git. Esse comando abre a URL da aplicação implantada para a branch ``sessions-in-redis``; a URL será semelhante a ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Teste o site nesse novo ambiente, você deve ver todos os dados que você criou no ambiente master.

Se você adicionar mais conferências no ambiente ``master``, elas não aparecerão no ambiente ``sessions-in-redis`` e vice-versa. Os ambientes são independentes e isolados.

Se o código evoluir no master, você sempre pode fazer o rebase da branch do Git e implantar a versão atualizada, resolvendo os conflitos tanto para o código quanto para a infraestrutura.

.. index::
    single: Symfony CLI;env:sync

Você pode até mesmo sincronizar os dados do master de volta para o ambiente ``sessions-in-redis``:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Depurando Implantações de Produção antes da Implantação
-------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Por padrão, todos os ambientes SymfonyCloud usam as mesmas configurações do ambiente ``master``/``prod`` (também conhecido como ambiente ``prod`` do Symfony). Isso permite que você teste a aplicação em condições reais. Isso lhe dá a sensação de desenvolver e testar diretamente em servidores de produção, mas sem os riscos associados. Isso me lembra os bons velhos tempos quando fazíamos a implantação via FTP.

.. index::
    single: Symfony CLI;env:debug

Em caso de problema, você pode querer mudar para o ambiente ``dev`` do Symfony:

.. code-block:: bash

    $ symfony env:debug

Quando terminar, volte para as configurações de produção:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Nunca** habilite o ambiente ``dev`` e nunca habilite o Profiler do Symfony na branch ``master``; isso tornaria sua aplicação muito lenta e abriria muitas vulnerabilidades sérias de segurança.

Testando Implantações de Produção antes da Implantação
------------------------------------------------------------

Ter acesso à próxima versão do site com dados de produção abre muitas oportunidades: desde testes visuais de regressão até testes de desempenho. `O Blackfire <https://blackfire.io>`_ é a ferramenta perfeita para o trabalho.

Consulte a etapa sobre "Desempenho" para saber mais sobre como você pode usar o Blackfire para testar o seu código antes de implantar.

Fazendo Merge para Produção
-----------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Quando você estiver satisfeito com as alterações na branch, faça o merge do código e da infraestrutura de volta para a branch master do Git:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

E implante:

.. code-block:: bash

    $ symfony deploy

Ao implantar, apenas as alterações de código e infraestrutura são enviadas para a SymfonyCloud; os dados não são afetados de forma alguma.

Pondo as Coisas em Ordem
------------------------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Por fim, ponha as coisas em ordem removendo a branch Git e o ambiente SymfonyCloud:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Indo Além

    * `Branches no Git <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
