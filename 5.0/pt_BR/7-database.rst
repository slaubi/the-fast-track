Configurando um Banco de Dados
==============================

.. index::
    single: Database

O objetivo do site Livro de Visitas da Conferência é coletar feedback durante as conferências. Precisamos armazenar os comentários dos participantes da conferência em um armazenamento permanente.

Um comentário é melhor descrito por uma estrutura de dados fixa: um autor, seu e-mail, o texto do feedback e uma foto opcional. É o tipo de dado que pode ser melhor armazenado em um banco de dados relacional tradicional.

PostgreSQL é o banco de dados que usaremos.

Adicionando o PostgreSQL ao Docker Compose
------------------------------------------

.. index::
    single: Docker;PostgreSQL

Em nossa máquina local decidimos usar o Docker para gerenciar serviços. Crie um arquivo ``docker-compose.yaml`` e adicione o PostgreSQL como um serviço:

.. code-block:: yaml
    :caption: docker-compose.yaml
    :emphasize-lines: 4,5,10

    version: '3'

    services:
        database:
            image: postgres:13-alpine
            environment:
                POSTGRES_USER: main
                POSTGRES_PASSWORD: main
                POSTGRES_DB: main
            ports: [5432]

Isso irá instalar um servidor PostgreSQL na versão 11 e configurar algumas variáveis de ambiente que controlam o nome e as credenciais do banco de dados. Os valores não importam.

Nós também expomos a porta do PostgreSQL (``5432``) no container para o host local. Isso nos ajudará a acessar o banco de dados a partir da nossa máquina.

.. note::

    A extensão ``pdo_pgsql`` deve ter sido instalada quando o PHP foi configurado em uma etapa anterior.

Iniciando o Docker Compose
--------------------------

Inicie o Docker Compose em segundo plano (``-d``):

.. code-block:: bash

    $ docker-compose up -d

Espere um pouco para deixar o banco de dados iniciar e verifique se tudo está funcionando bem:

.. code-block:: bash
    :class: ignore

    $ docker-compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Se não houver containers em execução ou se a coluna ``State`` não exibir ``Up``, verifique os logs do Docker Compose:

.. code-block:: bash
    :class: ignore

    $ docker-compose logs

Acessando o Banco de Dados Local
--------------------------------

Usar o utilitário de linha de comando ``psql`` pode ser útil de tempos em tempos. Mas você precisa lembrar das credenciais e do nome do banco de dados. Menos óbvio, você também precisa saber a porta local usada pelo banco de dados na máquina. O Docker escolhe uma porta aleatória para que você possa trabalhar em mais de um projeto usando o PostgreSQL ao mesmo tempo (a porta local é parte da saída do ``docker-compose ps``).

Se você executar o ``psql`` através da CLI do Symfony, não precisará lembrar de nada.

A CLI do Symfony detecta automaticamente os serviços Docker em execução para o projeto e expõe as variáveis de ambiente que o ``psql`` precisa para conectar ao banco de dados.

.. index::
    single: Symfony CLI;run psql

Graças a estas convenções, acessar o banco de dados usando ``symfony run`` é muito mais fácil:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

.. note::

    If you don't have the ``psql`` binary on your local host, you can also run it via ``docker-compose``:

    .. code-block:: bash
        :class: ignore

        $ docker-compose exec database psql main

Adicionando o PostgreSQL à SymfonyCloud
----------------------------------------

.. index::
    single: SymfonyCloud;PostgreSQL

Para a infraestrutura de produção na SymfonyCloud, adicionar um serviço como o PostgreSQL deve ser feito no arquivo ``.symfony/services.yaml``, atualmente vazio:

.. code-block:: yaml
    :caption: .symfony/services.yaml

    db:
        type: postgresql:13
        disk: 1024
        size: S

O serviço ``db`` é um banco de dados PostgreSQL na versão 11 (como o Docker) que nós queremos provisionar em um pequeno container com 1GB de disco.

Também precisamos "vincular" o BD ao container da aplicação, que é descrito em `.symfony.cloud.yaml``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    relationships:
        database: "db:postgresql"

O serviço ``db`` do tipo ``postgresql`` é referenciado como ``database`` no container da aplicação.

O último passo é adicionar a extensão ``pdo_pgsql`` ao runtime do PHP:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :class: ignore

    runtime:
        extensions:
            - pdo_pgsql
            # other extensions here

Aqui está o diff completo das alterações em ``.symfony.cloud.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - pdo_pgsql
             - apcu
             - mbstring
             - sodium
    @@ -21,6 +22,9 @@ build:

     disk: 512

    +relationships:
    +    database: "db:postgresql"
    +
     web:
         locations:
             "/":

Faça o commit destas alterações e então implante novamente na SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configuring the database'
    $ symfony deploy

Acessando o Banco de Dados na SymfonyCloud
------------------------------------------

O PostgreSQL agora está executando localmente via Docker e em produção na SymfonyCloud.

Como acabamos de ver, executar ``symfony run psql`` automaticamente conecta ao banco de dados hospedado pelo Docker graças às variáveis de ambiente expostas pelo ``symfony run``.

.. index::
    single: SymfonyCloud;Tunnel
    single: Symfony CLI;tunnel:open
    single: Symfony CLI;tunnel:close
    single: Symfony CLI;run psql

Se você quiser conectar ao PostgreSQL hospedado nos containers de produção, você pode abrir um túnel SSH entre a máquina local e a infraestrutura da SymfonyCloud:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars

Por padrão, os serviços da SymfonyCloud não são expostos como variáveis de ambiente na máquina local. Isso deve ser feito explicitamente através da flag ``--expose-env-vars``. Por quê? Conectar ao banco de dados de produção é uma operação perigosa. Você pode mexer com dados *reais*. Exigir a flag é como confirmar que *é* isso o que você deseja fazer.

Agora, conecte-se ao banco de dados PostgreSQL remoto usando ``symfony run psql`` como antes:

.. code-block:: bash
    :class: ignore

    $ symfony run psql

Quando acabar, não se esqueça de fechar o túnel:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:close

.. tip::

    Para executar algumas consultas SQL no banco de dados de produção, em vez de obter um shell, você também pode usar o comando ``symfony sql``.

Expondo Variáveis de Ambiente
------------------------------

.. index::
    single: SymfonyCloud;Environment Variables
    single: Symfony CLI;var:export

O Docker Compose e a SymfonyCloud funcionam perfeitamente com o Symfony graças às variáveis de ambiente.

Verifique todas as variáveis de ambiente expostas pelo ``symfony`` executando ``symfony var:export``:

.. code-block:: bash
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=main
    PGUSER=main
    PGPASSWORD=main
    # ...

As variáveis de ambiente ``PG*`` são lidas pelo utilitário ``psql``. E as outras?

Quando um túnel é aberto para a SymfonyCloud com a flag ``--expose-env-vars``, o comando ``var:export`` retorna as variáveis de ambiente remotas:

.. code-block:: bash
    :class: ignore

    $ symfony tunnel:open --expose-env-vars
    $ symfony var:export
    $ symfony tunnel:close

.. sidebar:: Indo Além

    * `SymfonyCloud services <https://symfony.com/doc/current/cloud/services/intro.html#available-services>`_;

    * `SymfonyCloud tunnel <https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service>`_;

    * `Documentação do PostgreSQL <https://www.postgresql.org/docs/>`_;

    * `Comandos docker-compose <https://docs.docker.com/compose/reference/>`_.
