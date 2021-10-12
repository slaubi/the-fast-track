Adotando uma Metodologia
========================

Ensinar é repetir a mesma coisa várias vezes. Não vou fazer isso. Eu prometo. Ao final de cada etapa você deve fazer uma pequena dança e salvar o seu trabalho. É como ``Ctrl+S``, mas para um site.

Implementando uma Estratégia Git
---------------------------------

.. index::
    single: Git;add
    single: Git;commit

Ao final de cada passo não se esqueça de fazer o commit das suas alterações:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Você pode adicionar "tudo" com segurança, pois o Symfony gerencia um arquivo ``.gitignore`` para você. E cada pacote pode adicionar mais configurações. Dê uma olhada no conteúdo atual:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,8

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

As strings engraçadas são marcadores adicionados pelo Symfony Flex para que ele saiba o que remover se você decidir desinstalar uma dependência. Eu te disse, todo o trabalho tedioso é feito pelo Symfony, não por você.

Seria bom fazer um push do seu repositório para um servidor em algum lugar. GitHub, GitLab ou Bitbucket são boas escolhas.

Se você estiver implantando na SymfonyCloud, você já tem uma cópia do repositório Git, mas não deve confiar nela. É apenas para uso de implantação. Não é um backup.

Implantação Contínua em Produção
-------------------------------------

.. index::
    single: Symfony CLI;deploy

Outro bom hábito é a implantação frequente. A implantação no final de cada etapa é um bom ritmo:

.. code-block:: bash
    :class: ignore

    $ symfony deploy
