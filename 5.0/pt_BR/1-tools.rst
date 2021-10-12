Verificando Seu Ambiente de Trabalho
====================================

Antes de começar a trabalhar no projeto, precisamos verificar se todos têm um bom ambiente de trabalho. Isso é muito importante. As ferramentas de desenvolvimento que temos hoje à nossa disposição são muito diferentes das que tínhamos há 10 anos. Elas evoluíram muito, para melhor. Seria uma pena não aproveitá-las. Boas ferramentas podem te levar ainda mais longe.

Por favor, não pule esta etapa. Ou pelo menos, leia a última seção sobre a CLI do Symfony.

Um Computador
-------------

Você precisa de um computador. A boa notícia é que ele pode ter qualquer SO popular instalado: macOS, Windows ou Linux. Symfony e todas as ferramentas que vamos utilizar são compatíveis com cada um deles.

Escolhas Pessoais
-----------------

Quero avançar rapidamente com as melhores opções disponíveis. Fiz escolhas baseadas no que acredito ser o melhor para esse livro.

O `PostgreSQL`_ será nossa escolha para banco de dados.

`RabbitMQ`_ é o vencedor para as filas.

IDE
---

.. index:: IDE

Você pode usar o Notepad se quiser. Mas, eu não o recomendaria.

Eu costumava trabalhar com o Textmate. Não mais. O conforto de usar uma IDE "real" é inestimável. Auto-completar, instruções ``use`` adicionadas e ordenadas automaticamente, saltar de um arquivo para outro são alguns dos recursos que aumentarão sua produtividade.

Eu recomendaria usar o `Visual Studio Code`_ ou `PhpStorm`_. O primeiro é livre, o segundo não é, mas tem uma melhor integração com o Symfony (graças ao `Plugin Symfony Support`_). Vai de você. Sei que quer saber que IDE estou usando. Estou escrevendo este livro no Visual Studio Code.

Terminal
--------

.. index:: Terminal

Vamos mudar da IDE para a linha de comando o tempo todo. Você pode usar o terminal embutido de sua IDE, mas eu prefiro usar um terminal real para ter mais espaço.

O Linux vem com o ``Terminal`` embutido. Utilize o `iTerm2`_ no macOS. No Windows, o `Hyper`_ funciona bem.

Git
---

.. index:: Git

O meu último livro recomendou o Subversion para controle de versões. Parece que todos estão usando o `Git`_ agora.

No Windows, instale o `Git bash`_.

Certifique-se de que você sabe como executar as operações comuns como ``git clone``, ``git log``, ``git show``, ``git diff``, ``git checkout``, ...

PHP
---

.. index:: PHP

Vamos usar o Docker para serviços, mas eu gosto de ter o PHP instalado no meu computador local por razões de performance, estabilidade e simplicidade. Me chame de antiquado se quiser, mas a combinação de um PHP local e serviços Docker é a combinação perfeita para mim.

Use o PHP 7.3 se puder, talvez 7.4 dependendo de quando você estiver lendo este livro. Verifique se as :index:`extensões PHP a seguir <Extensões PHP>` estão instaladas ou instale-as agora: ``intl``, ``pdo_pgsql``, ``xsl``, ``amqp``, ``gd``, ``openssl``, ``sodium``.  Opcionalmente, instale ``redis`` e ``curl`` também.

Você pode verificar as extensões atualmente habilitadas via ``php -m``.

Também precisamos do ``php-fpm`` se a sua plataforma suportar, mas ``php-cgi`` funciona também.

Composer
--------

.. index:: Composer

Gerenciar dependências é tudo hoje em dia com um projeto Symfony. Obtenha a versão mais recente do `Composer <https://getcomposer.org/>`_, a ferramenta de gerenciamento de pacotes para PHP.

Se você não estiver familiarizado com o Composer, reserve algum tempo para ler sobre ele.

.. tip::

    Você não precisa digitar os nomes completos do comando: ``composer req`` faz o mesmo que ``composer require``, use ``composer rem`` em vez de ``composer remove``, ...

Docker e Docker Compose
-----------------------

.. index:: Docker,Docker Compose

Os serviços serão gerenciados pelo Docker e pelo Docker Compose. `Instale-os <https://docs.docker.com/install/>`_ e inicie o Docker. Se você estiver utilizando o Docker pela primeira vez, familiarize-se com a ferramenta antes. Mas não entre em pânico, a nossa utilização será muito simples. Sem configurações extravagantes e complexas.

Symfony CLI
-----------

.. index:: Symfony CLI

Por último, mas não menos importante, vamos usar a CLI ``symfony`` para aumentar a nossa produtividade. Desde o servidor web local que ela fornece, até a integração total com o Docker e o suporte à SymfonyCloud, será uma grande economia de tempo.

Instale a `CLI do Symfony <https://symfony.com/download>`_ e adicione-a ao seu ``$PATH``. Crie uma conta no `SymfonyConnect <https://connect.symfony.com/>`_ se você ainda não tiver e faça o login via ``symfony login``.

Para usar HTTPS localmente, também precisamos `instalar um CA <https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls>`_ para habilitar o suporte a TLS. Execute o seguinte comando:

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: bash
    :class: ignore

    $ symfony server:ca:install

Verifique se o seu computador tem todos os requisitos necessários executando o seguinte comando:

.. code-block:: bash
    :class: ignore

    $ symfony book:check-requirements

Se você quiser ser sofisticado, você também pode executar o `proxy do Symfony <https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy>`_. É opcional, mas permite que você obtenha um domínio local terminando com ``.wip`` para o seu projeto.

Ao executar um comando em um terminal, iremos quase sempre prefixá-lo com ``symfony`` como em ``symfony composer`` em vez de apenas ``composer``, ou ``symfony console`` em vez de ``./bin/console``.

A principal razão é que a CLI do Symfony define automaticamente algumas variáveis de ambiente com base nos serviços em execução em sua máquina via Docker. Essas variáveis de ambiente estão disponíveis para requisições HTTP porque o servidor web local as injeta automaticamente. Assim, usar ``symfony`` na CLI garante que você tenha o mesmo comportamento de maneira geral.

Além disso, a CLI do Symfony seleciona automaticamente a "melhor" versão PHP possível para o projeto.

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Plugin Symfony Support`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
