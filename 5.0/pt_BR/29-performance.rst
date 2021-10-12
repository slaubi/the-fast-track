Lidando com o Desempenho
========================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    Otimização prematura é a raiz de todo o mal.

Talvez você já tenha lido esta citação antes. Mas gosto de citá-la na íntegra:

.. epigraph::

    Devemos esquecer as pequenas eficiências, digamos, 97% das vezes: a otimização prematura é a raiz de todo o mal. No entanto, não devemos desperdiçar nossas oportunidades naqueles 3% críticos.

    --   Donald Knuth

Mesmo pequenas melhorias de desempenho podem fazer a diferença, especialmente para sites de e-commerce. Agora que a aplicação do livro de visitas está pronta para o horário nobre, vamos ver como podemos verificar o seu desempenho.

A melhor maneira de encontrar otimizações de desempenho é usar um *profiler*. A opção mais popular hoje em dia é o `Blackfire <https://blackfire.io>`_ (*aviso*: eu também sou o fundador do projeto Blackfire).

Apresentamos o Blackfire
------------------------

O Blackfire é composto de várias partes:

* Um *cliente* que aciona profiles (a Ferramenta CLI do Blackfire ou uma extensão de navegador para Google Chrome ou Firefox);

* Um *agente* que prepara e agrega dados antes de enviá-los para blackfire.io para exibição;

* Uma extensão PHP (a *sonda*) que instrumenta o código PHP.

Para trabalhar com o Blackfire, você precisa primeiro `criar uma conta <https://blackfire.io/signup>`_.

Instale o Blackfire em sua máquina local executando o seguinte script de instalação rápida:

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Este instalador baixa a Ferramenta CLI do Blackfire e depois instala a sonda PHP (sem ativá-la) em todas as versões disponíveis do PHP.

Habilite a sonda PHP para nosso projeto:

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Reinicie o servidor web para que o PHP possa carregar o Blackfire:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

A Ferramenta CLI do Blackfire precisa ser configurada com suas credenciais pessoais de **cliente** (para armazenar os profiles de seu projeto em sua conta pessoal). Encontre-as no topo da `página <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials`` e execute o seguinte comando substituindo os espaços reservados:

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Para obter instruções completas de instalação, siga o `guia de instalação detalhada oficial <https://blackfire.io/docs/up-and-running/installation>`_. Elas são úteis ao instalar o Blackfire em um servidor.

Configurando o Agente do Blackfire no Docker
--------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

A última etapa é adicionar o serviço do agente do Blackfire na stack do Docker Compose:

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -20,3 +20,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Para se comunicar com o servidor, você precisa obter suas credenciais pessoais de **servidor** (essas credenciais identificam onde você deseja armazenar os profiles - você pode criar uma por projeto); elas podem ser encontradas na parte inferior da `página <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials``. Armazene-as em um arquivo local ``.env.local``:

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Você agora pode iniciar o novo container:

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Corrigindo uma Instalação do Blackfire Que não Funciona
----------------------------------------------------------

Se você receber um erro ao fazer profiles, aumente o nível de log do Blackfire para obter mais informações nos logs:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Reinicie o servidor web:

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

E verifique os logs mais recentes:

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Faça o profile novamente e verifique a saída nos logs.


Configurando o Blackfire em Produção
--------------------------------------

.. index::
    single: SymfonyCloud;Blackfire

O Blackfire é incluído por padrão em todos os projetos da SymfonyCloud.

Configure as credenciais do *servidor* como variáveis de ambiente:

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

E habilite a sonda PHP como qualquer outra extensão PHP:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - amqp
             - redis

Configurando o Varnish para o Blackfire
---------------------------------------

.. index::
    single: SymfonyCloud;Varnish

Antes que você possa implantar para começar a fazer profiles, você precisa de uma maneira de ignorar o cache HTTP do Varnish. Caso contrário, o Blackfire nunca irá atingir a aplicação PHP. Você vai autorizar apenas requisições de profiles vindo da sua máquina local.

Encontre o seu endereço IP atual:

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

E utilize-o para configurar o Varnish:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

Agora você pode implantar.

Fazendo Profiles de Páginas Web
--------------------------------

.. index::
    single: Profiling;Web Pages

Você pode fazer profiles de páginas web tradicionais a partir do Firefox ou do Google Chrome por meio de suas `extensões dedicadas <https://blackfire.io/docs/integrations/browsers/index>`_.

Em sua máquina local, não se esqueça de desabilitar o cache HTTP em ``public/index.php`` ao fazer profiles: caso contrário, você irá fazer o profile da camada de cache HTTP do Symfony em vez do seu próprio código:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/public/index.php
    +++ b/public/index.php
    @@ -24,7 +24,7 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? $_ENV['TRUSTED_HOSTS'] ?? false
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);

     if ('dev' === $kernel->getEnvironment()) {
    -    $kernel = new HttpCache($kernel);
    +//    $kernel = new HttpCache($kernel);
     }

     $request = Request::createFromGlobals();

Para ter uma ideia melhor do desempenho da sua aplicação em produção, você também deve fazer profiles do ambiente de "produção". Por padrão, seu ambiente local usa o ambiente de "desenvolvimento", que adiciona uma sobrecarga significativa (principalmente para coletar dados para a barra de ferramentas para depuração web e para o Profiler do Symfony).

.. index::
    single: Symfony CLI;server:prod

Mudar a sua máquina local para o ambiente de produção pode ser feito alterando a variável de ambiente ``APP_ENV`` no arquivo ``.env.local``:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Ou você pode usar o comando ``server:prod``:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

Não esqueça de mudar de volta para o ambiente de desenvolvimento quando você terminar de fazer profiles:

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Fazendo Profiles de Recursos da API
-----------------------------------

.. index::
    single: Profiling;API

É melhor fazer profiles da API ou do SPA pela CLI através da Ferramenta CLI do Blackfire que você instalou anteriormente:

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

O comando ``blackfire curl`` aceita exatamente os mesmos argumentos e opções que o `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Comparando o Desempenho
-----------------------

Na etapa sobre "Cache", adicionamos uma camada de cache para melhorar o desempenho do nosso código, mas não verificamos nem medimos o impacto desta mudança no desempenho. Como todos nós somos muito ruins em adivinhar o que será rápido e o que é lento, você pode acabar em uma situação na qual otimizar alguma coisa na verdade torna sua aplicação mais lenta.

Você sempre deve medir o impacto de qualquer otimização que você fizer com um profiler. O Blackfire torna isso mais fácil visualmente graças ao seu `recurso de comparação <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Escrevendo Testes Funcionais Caixa Preta
----------------------------------------

.. index::
    single: Blackfire;Player

Já vimos como escrever testes funcionais com o Symfony. O Blackfire pode ser usado para escrever cenários de navegação que podem ser executados sob demanda através do `player do Blackfire <https://blackfire.io/player>`_. Vamos escrever um cenário onde um novo comentário é submetido e validado através do link de e-mail no ambiente de desenvolvimento e através do painel administrativo em produção.

Crie um arquivo ``.blackfire.yaml`` com o seguinte conteúdo:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Baixe o player do Blackfire para poder executar o cenário localmente:

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Execute este cenário no ambiente de desenvolvimento:

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Ou em produção:

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Os cenários do Blackfire podem também acionar profiles a cada requisição e executar testes de desempenho adicionando a flag ``--blackfire``.

Automatizando Verificações de Desempenho
------------------------------------------

Gerenciar o desempenho não é apenas melhorar o desempenho do código existente, mas também verificar se nenhuma regressão de desempenho foi introduzida.

O cenário escrito na seção anterior pode ser executado automaticamente em um workflow de integração contínua ou periodicamente em produção.

Na SymfonyCloud, também é possível `executar cenários <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ sempre que você criar uma nova branch ou implantar em produção para verificar automaticamente o desempenho do novo código.

.. sidebar:: Indo Além

    * `O livro do Blackfire: PHP Code Performance Explained <https://blackfire.io/book>`_;

    * `Tutorial do Blackfire no SymfonyCasts <https://symfonycasts.com/screencast/blackfire>`_.
