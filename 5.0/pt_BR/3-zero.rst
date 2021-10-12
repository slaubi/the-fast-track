Indo do Zero à Produção
==========================

Gosto de ir rápido. Quero que o nosso pequeno projeto vá ao ar o mais rápido possível. Digo, agora. Em produção. Como ainda não desenvolvemos nada, vamos começar desenvolvendo uma simples e agradável página "Em construção". Você vai adorar!

Passe algum tempo tentando encontrar a GIF "Em construção" ideal, clássica e animada na Internet. `Aqui está <http://clipartmag.com/images/website-under-construction-image-6.gif>`_ a que eu vou usar:

.. image:: images/under-construction.gif
    :align: center

Eu te disse, vai ser muito divertido.

Inicializando o Projeto
-----------------------

Crie um novo projeto Symfony com a ferramenta CLI ``symfony`` que instalamos juntos anteriormente:

.. code-block:: bash

    $ symfony new guestbook --version=5.0
    $ cd guestbook

Este comando coloca uma camada fina sobre o ``Composer`` que facilita a criação de projetos Symfony. Ele usa um `esqueleto de projeto <https://github.com/symfony/skeleton>`_ que inclui as dependências fundamentais; os componentes do Symfony que são necessários para quase qualquer projeto: uma ferramenta de console e a abstração HTTP que permite criar aplicações Web.

Se você der uma olhada no repositório do GitHub para o esqueleto, você notará que ele está quase vazio. Contém apenas um arquivo ``composer.json``. Mas o diretório ``guestbook`` está cheio de arquivos. Como é que isso é possível? A resposta está no pacote ``symfony/flex``. O Symfony Flex é um plugin do Composer que se conecta ao processo de instalação. Ao identificar um pacote para o qual há uma *receita*, ele a executa.

O principal ponto de entrada de uma Receita do Symfony é um arquivo de manifesto que descreve as operações que precisam ser feitas para registrar automaticamente o pacote em uma aplicação Symfony. Você nunca precisa ler um arquivo README para instalar um pacote com o Symfony. A automação é uma característica essencial do Symfony.

Como o Git está instalado na nossa máquina, ``symfony new`` também criou um repositório Git para nós e adicionou o primeiro commit.

Dê uma olhada na estrutura de diretórios:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

O diretório ``bin/``  contém o ponto de entrada principal da CLI: ``console``. Você irá usá-lo o tempo todo.

O diretório ``config/``  é composto por um conjunto padrão e sensato de arquivos de configuração. Um arquivo por pacote. Você dificilmente irá mudá-los, confiar nos padrões é quase sempre uma boa idéia.

O diretório ``public/``  é o diretório raiz da web, e o script ``index.php`` é o ponto de entrada principal para todos os recursos HTTP dinâmicos.

É no diretório ``src/`` que ficará todo o código que você irá escrever; é onde você passará a maior parte do seu tempo. Por padrão, todas as classes neste diretório usam o namespace ``App``. É a sua casa. Seu código. A sua lógica de domínio. O Symfony tem muito pouca influência aqui.

O diretório ``var/`` contém caches, logs e arquivos gerados em tempo de execução pela aplicação. Você pode deixá-lo em paz. É o único diretório que precisa ter permissão de escrita no ambiente de produção.

O diretório ``vendor/`` contém todos os pacotes instalados pelo Composer, incluindo o próprio Symfony. Essa é a nossa arma secreta para sermos mais produtivos. Não vamos reinventar a roda. Você vai confiar nas bibliotecas existentes para fazer o trabalho duro. O diretório é gerenciado pelo Composer. Nunca altere esse diretório.

Isso é tudo o que você precisa saber por enquanto.

Criando Alguns Recursos Públicos
---------------------------------

Qualquer coisa em ``public/`` é acessível através de um navegador. Por exemplo, se você mover seu GIF animado (renomeie-o como ``under-construction.gif``) para um novo diretório ``public/images/``, ele estará disponível em uma URL como ``https://localhost/images/under-construction.gif``.

Baixe minha imagem GIF aqui:

.. code-block:: bash

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Iniciando um Servidor Web Local
-------------------------------

.. index::
    single: Symfony CLI;server:start

A CLI ``symfony`` vem com um Servidor Web que é otimizado para o trabalho de desenvolvimento. Você não ficará surpreso se eu disser que ele funciona muito bem com o Symfony. Mas nunca o use em produção.

No diretório do projeto, inicie o servidor web em segundo plano (flag ``-d``):

.. code-block:: bash

    $ symfony server:start -d

O servidor inicia na primeira porta disponível, a partir da porta 8000. Como um atalho, abra o site em um navegador a partir da CLI:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

Seu navegador padrão deve abrir uma nova aba que exibe algo semelhante ao seguinte:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    Para solucionar problemas, execute ``symfony server:log``; isto agrupa os logs do servidor web, do PHP e da sua aplicação.

Navegue até ``/images/under-construction.gif``. Se parece com isto?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

Curtiu? Vamos fazer o commit do nosso trabalho:

.. code-block:: bash
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Adicionando um favicon
----------------------

Para evitar ser "bombardeado" por erros HTTP 404 nos logs devido à falta de um favicon requisitado pelos navegadores, vamos adicionar um agora:

.. code-block:: bash

    $ php -r "copy('https://symfony.com/favicon.ico', 'public/favicon.ico');"
    $ git add public/
    $ git commit -m'Add a favicon'

Preparando para Produção
--------------------------

.. index::
    single: SymfonyCloud;Initialization

Que tal implantar o nosso trabalho em produção? Eu sei, ainda não temos sequer uma página HTML adequada para dar as boas-vindas aos nossos usuários. Mas ser capaz de ver a pequena imagem "em construção" em um servidor de produção seria um grande avanço. E você conhece o lema: *publique cedo e com frequência*.

Você pode hospedar esta aplicação em qualquer provedor que suporte PHP... o que significa praticamente todos os provedores de hospedagem que existem. Mas verifique algumas coisas: queremos a última versão do PHP e a possibilidade de hospedar serviços como um banco de dados, uma fila, e algo mais.

Fiz a minha escolha e será a `SymfonyCloud <https://symfony.com/cloud>`_. Ela fornece tudo o que precisamos e ajuda a financiar o desenvolvimento do Symfony.

.. index::
    single: Symfony CLI;project:init

A CLI ``symfony`` tem suporte nativo à SymfonyCloud. Vamos inicializar um projeto SymfonyCloud:

.. code-block:: bash

    $ symfony project:init

Este comando cria alguns arquivos necessários para a SymfonyCloud, especificamente ``.symfony/services.yaml``, ``.symfony/routes.yaml`` e ``.symfony.cloud.yaml``.

Adicione-os ao Git e faça um commit:

.. code-block:: bash

    $ git add .
    $ git commit -m"Add SymfonyCloud configuration"

.. note::

    Usar o genérico e perigoso ``git add .`` funciona muito bem, já que foi gerado um arquivo ``.gitignore`` que ignora automaticamente todos os arquivos que não queremos adicionar ao commit.

Indo para Produção
--------------------

.. index::
    single: Symfony CLI;project:create
    single: Symfony CLI;deploy

Hora de implantar?

Crie um novo Projeto SymfonyCloud:

.. code-block:: bash

    $ symfony project:create --title="Guestbook" --plan=development

Este comando faz muita coisa:

* A primeira vez que você executar este comando, autentique-se com suas credenciais SymfonyConnect caso já não o tenha feito.

* Ele provisiona um novo projeto na SymfonyCloud (você ganha 7 dias *de graça* em qualquer novo projeto de desenvolvimento).

Então, implante:

.. code-block:: bash

    $ symfony deploy

A implantação é feita ao fazer o push do repositório Git. No final do comando, o projeto terá um domínio específico que você pode usar para acessá-lo.

.. index::
    single: Symfony CLI;open:remote

Veja se tudo funcionou bem:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Você deve receber um 404, mas navegar para ``/images/under-construction.gif`` deve revelar o nosso trabalho.

Observe que você não verá a bela página padrão do Symfony na SymfonyCloud. Por quê? Você aprenderá em breve que o Symfony suporta vários ambientes e que a SymfonyCloud implantou o código automaticamente no ambiente de produção.

.. index::
    single: Symfony CLI;project:delete

.. tip::

    Se você quiser excluir o projeto na SymfonyCloud, use o comando ``project:delete``.

.. sidebar:: Indo Além

    * O `Servidor de Receitas do Symfony <https://flex.symfony.com/>`_, onde você pode encontrar todas as receitas disponíveis para suas aplicações Symfony;

    * Os repositórios para as `receitas oficiais do Symfony <https://github.com/symfony/recipes>`_ e para as `receitas disponibilizadas pela comunidade <https://github.com/symfony/recipes-contrib>`_, onde você pode enviar suas próprias receitas;

    * O `Servidor Web Local do Symfony <https://symfony.com/doc/current/setup/symfony_server.html>`_;

    * A `documentação da SymfonyCloud <https://symfony.com/doc/cloud>`_.
