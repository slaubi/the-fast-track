Resolvendo Problemas
====================

Configurar um projeto significa também ter as ferramentas certas para depurar problemas.

Instalando mais Dependências
-----------------------------

Lembre-se que o projeto foi criado com muito poucas dependências. Sem mecanismo de templates. Sem ferramentas de depuração. Sem sistema de logs. A idéia é que você pode adicionar mais dependências sempre que precisar. Por que você dependeria de um mecanismo de templates se você desenvolver uma API HTTP ou uma ferramenta de CLI?

Como podemos adicionar mais dependências? Via Composer. Além dos pacotes "regulares" do Composer, trabalharemos com dois tipos "especiais" de pacotes:

* *Componentes Symfony*: Pacotes que implementam recursos centrais e abstrações de baixo nível que a maioria das aplicações precisa (roteamento, console, cliente HTTP, mailer, cache, ...);

* *Symfony Bundles*: Pacotes que adicionam recursos de alto nível ou fornecem integrações com bibliotecas de terceiros (os bundles são criados principalmente pela comunidade).

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Para começar, vamos adicionar o Profiler do Symfony, um recurso eficiente para encontrar a causa raiz de um problema:

.. code-block:: bash

    $ symfony composer req profiler --dev

``profiler`` é um alias para o pacote ``symfony/profiler-pack``.

*Aliases* não são um recurso do Composer, mas um conceito fornecido pelo Symfony para facilitar sua vida. Os aliases são atalhos para pacotes populares do Composer. Quer um ORM para sua aplicação? Requisite um ``orm``. Quer desenvolver uma API? Requisite um ``api``. Esses aliases são automaticamente convertidos para um ou mais pacotes regulares do Composer. Eles são escolhas feitas pela equipe central do Symfony.

Outro recurso interessante é que você sempre pode omitir o vendor ``symfony``. Requisite ``cache`` em vez de ``symfony/cache``.

.. tip::

    Você se lembra que mencionamos um plugin do Composer chamado ``symfony/flex`` antes? Os aliases são um dos seus recursos.

Entendendo os Ambientes do Symfony
----------------------------------

Notou a flag ``--dev`` no comando ``composer req``? Como o Profiler do Symfony é útil apenas durante o desenvolvimento, queremos evitar que ele seja instalado em produção.

O Symfony suporta a criação de *ambientes*. Por padrão, ele tem suporte nativo a três, mas você pode adicionar quantos quiser: ``dev``, ``prod`` e ``test``.  Todos os ambientes compartilham o mesmo código, mas representam *configurações* diferentes.

Por exemplo, todas as ferramentas de depuração estão habilitadas no ambiente ``dev``. No ambiente ``prod`` a aplicação é otimizada para desempenho.

Mudar de um ambiente para outro pode ser feito alterando a variável de ambiente ``APP_ENV``.

Quando você implantou na SymfonyCloud, o ambiente (armazenado em ``APP_ENV``) foi alterado automaticamente para ``prod``.

Gerenciando Configurações de Ambiente
---------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` pode ser definida usando variáveis de ambiente "reais" no seu terminal:

.. code-block:: bash
    :class: ignore

    $ export APP_ENV=dev

Usar variáveis de ambiente reais é a forma preferida de definir valores como ``APP_ENV`` nos servidores de produção. Mas nas máquinas de desenvolvimento, ter que definir muitas variáveis de ambiente pode ser complicado. Em vez disso, defina-as em um um arquivo ``.env``.

Um arquivo ``.env`` razoável foi gerado automaticamente para você quando o projeto foi criado:

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    Qualquer pacote pode adicionar mais variáveis de ambiente a este arquivo graças à sua receita usada pelo Symfony Flex.

O arquivo ``.env`` é armazenado no repositório e descreve os valores *padrão* do ambiente de produção. É possível substituir esses valores criando um arquivo ``.env.local``. Este arquivo não deve ser armazenado no repositório e é por isso que o arquivo ``.gitignore`` já o ignora.

Nunca armazene valores secretos ou sensíveis nestes arquivos. Veremos como gerenciar dados sensíveis mais à frente.

Registrando Tudo no Log
-----------------------

.. index::
    single: Logger

De início, as capacidades de log e depuração são limitadas em novos projetos. Vamos adicionar mais ferramentas para nos ajudar a investigar problemas no desenvolvimento e também em produção:

.. code-block:: bash

    $ symfony composer req logger

.. index::
    single: Components;Debug
    single: Debug

Vamos instalar as ferramentas de depuração apenas em ambiente de desenvolvimento:

.. code-block:: bash

    $ symfony composer req debug --dev

Descobrindo as Ferramentas de Depuração do Symfony
----------------------------------------------------

Se você atualizar a página inicial, você deve agora ver uma barra de ferramentas na parte inferior da tela:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

A primeira coisa que você pode notar é o **404** em vermelho. Lembre-se que esta página é um placeholder, pois ainda não definimos uma página inicial. Mesmo que a página padrão que lhe dá as boas vindas seja bonita, ela ainda é uma página de erro. Então o código de status HTTP correto é 404, não 200. Graças à barra de ferramentas para depuração web, você tem estas informações imediatamente.

Se você clicar no pequeno ponto de exclamação, você obtém a "verdadeira" mensagem de exceção como parte dos logs no Profiler do Symfony. Se você quiser ver o stack trace, clique no link "Exception" no menu esquerdo.

Sempre que houver um problema com o seu código você verá uma página de exceção como a seguinte, que lhe dá tudo o que você precisa para entender o problema e de onde ele vem:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Dedique algum tempo para explorar as informações dentro do Profiler do Symfony clicando nos links disponíveis.

.. index::
    single: Symfony CLI;server:log

Logs também são bastante úteis em sessões de depuração. O Symfony tem um comando conveniente para exibir todos os logs (do servidor web, do PHP e da sua aplicação):

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Vamos fazer uma pequena experiência. Abra ``public/index.php`` e quebre o código PHP lá (adicione foobar no meio do código, por exemplo). Atualize a página no navegador e observe o fluxo de logs:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

A saída é lindamente colorida para chamar a sua atenção para os erros.

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Outra grande ajuda na depuração é a função ``dump()`` do Symfony. Ela está sempre disponível e permite que você visualize variáveis complexas em um formato agradável e interativo.

Modifique temporariamente o arquivo ``public/index.php`` para exibir o objeto Request:

.. code-block:: diff
    :caption: patch_file

    --- a/public/index.php
    +++ b/public/index.php
    @@ -23,5 +23,8 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? false) {
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
     $request = Request::createFromGlobals();
     $response = $kernel->handle($request);
    +
    +dump($request);
    +
     $response->send();
     $kernel->terminate($request, $response);

Ao atualizar a página observe o novo ícone de "alvo" na barra de ferramentas; ele permite que você inspecione os detalhes do objeto. Clique nele para acessar uma página inteira onde a navegação é simplificada:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Reverta as alterações antes de fazer o commit das outras alterações feitas nesta etapa:

.. code-block:: bash

    $ git checkout public/index.php

Configurando Sua IDE
--------------------

No ambiente de desenvolvimento, quando uma exceção é lançada, o Symfony exibe uma página com a mensagem da exceção e seu stack trace. Ao exibir um caminho de arquivo, ele adiciona um link que abre o arquivo na linha correta na sua IDE favorita. Para se beneficiar deste recurso você precisa configurar sua IDE. O Symfony suporta muitas IDEs sem necessidade de configuração; estou usando o Visual Studio Code para este projeto:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -6,3 +6,4 @@ max_execution_time=30
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

Os arquivos com links não estão limitados a exceções. Por exemplo, o controlador na barra de ferramentas para depuração web torna-se clicável após a configuração da IDE.

Depurando em Produção
-----------------------

.. index::
    single: SymfonyCloud;Remote Logs
    single: SymfonyCloud;SSH
    single: Symfony CLI;logs
    single: Symfony CLI;ssh

A depuração nos servidores de produção é sempre mais complicada. Você não tem acesso ao Profiler do Symfony, por exemplo. Os logs são menos verbosos. Mas visualizar os logs é possível:

.. code-block:: bash
    :class: ignore

    $ symfony logs

Você pode até mesmo se conectar via SSH ao container web:

.. code-block:: bash
    :class: ignore

    $ symfony ssh

Não se preocupe, você não consegue quebrar nada facilmente. A maior parte do sistema de arquivos é somente leitura. Não será possível fazer uma correção diretamente em produção. Mas você aprenderá uma maneira muito melhor mais tarde no livro.

.. sidebar:: Indo Além

    * `SymfonyCasts: tutorial sobre ambientes e arquivos de configuração <https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files>`_;

    * `SymfonyCasts: tutorial sobre variáveis de ambiente <https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables>`_;

    * `SymfonyCasts: tutorial sobre a Barra de Ferramentas para Depuração Web e o Profiler <https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler>`_;

    * `Gerenciando vários arquivos .env <https://symfony.com/doc/current/configuration.html#managing-multiple-env-files>`_ em aplicações Symfony.
