Apresentando o Projeto
======================

Precisamos encontrar um projeto para trabalhar. É um grande desafio, pois precisamos encontrar um projeto grande o suficiente para cobrir todo o Symfony mas, ao mesmo tempo, ele deve ser pequeno o suficiente; não quero que você fique entediado implementando recursos similares mais de uma vez.

Revelando o Projeto
-------------------

Como o livro tem que ser lançado durante a SymfonyCon Amsterdam, pode ser legal se o projeto for relacionado ao Symfony e a conferências. Que tal um `livro de visitas <https://pt.wikipedia.org/wiki/Livro_de_visitas>`_? Um livre d'or, como dizemos em francês. Eu gosto do sentimento antiquado e desatualizado de desenvolver um livro de visitas em 2019!

Já temos o projeto. Ele tem como objetivo obter feedback sobre as conferências: uma lista de conferências na página inicial, uma página para cada conferência, cheia de comentários simpáticos. Um comentário é composto por um pequeno texto e uma fotografia opcional tirada durante a conferência. Suponho que acabei de escrever todas as especificações que precisamos para começar.

O *projeto* conterá várias *aplicações*. Uma aplicação web tradicional com um frontend HTML, uma API e um SPA para telefones celulares. O que acha?

Aprender É Fazer
-----------------

Aprender é fazer. Ponto final. Ler um livro sobre o Symfony é bom. Programar uma aplicação no seu computador pessoal enquanto lê um livro sobre Symfony é ainda melhor. Este livro é muito especial pois tudo foi feito para permitir que você acompanhe, codifique, e tenha os mesmos resultados que eu tive localmente na minha máquina quando eu a programei inicialmente.

O livro contém todo o código que você precisa escrever e todos os comandos que você precisa executar para obter o resultado final. Não falta nenhum código. Todos os comandos estão contidos aqui. Isto é possível porque as aplicações Symfony modernas têm muito pouco código boilerplate. A maior parte do código que vamos escrever juntos é sobre a *lógica de negócio* do projeto. Todo o resto é automatizado ou gerado automaticamente para nós.

Olhando o Diagrama Final da Infraestrutura
------------------------------------------

Mesmo que a idéia do projeto pareça simples, não vamos construir um projeto do tipo "Hello World". Não usaremos apenas PHP e um banco de dados.

O objetivo é criar um projeto com algumas das complexidades que você pode encontrar na vida real. Quer uma prova? Dê uma olhada na infraestrutura final do projeto:

.. figure:: images/infrastructure.svg
    :align: center
    :figclass: ad diagram

Um dos grandes benefícios de usar um framework é a pequena quantidade de código necessária para desenvolver tal projeto:

* 20 classes PHP em ``src/`` para o site;

* 550 Linhas de Código Lógico (LLOC) PHP conforme reportado pelo `PHPLOC <https://github.com/sebastianbergmann/phploc>`_;

* 40 linhas de configuração em 3 arquivos (via annotations e YAML), principalmente para configurar o design do backend;

* 20 linhas de configuração da infraestrutura de desenvolvimento (Docker);

* 100 linhas de configuração da infraestrutura de produção (SymfonyCloud);

* 5 variáveis de ambiente explícitas.

Pronto para o desafio?

Obtendo o Código-Fonte do Projeto
----------------------------------

Para continuar com o tema antiquado, eu poderia ter gravado um CD contendo o código-fonte, certo? Mas que tal a companhia de um repositório Git em vez disso?

.. index::
    single: Project;Git Repository
    single: Git;clone

Clone o `repositório do livro de visitas <https://github.com/the-fast-track/book-5.0-6>`_ em algum lugar na sua máquina local:

.. code-block:: bash
    :class: ignore

    $ symfony new --version=5.0-6 --book guestbook

Este repositório contém todo o código do livro.

Note que estamos usando ``symfony new`` ao invés de ``git clone``, pois o comando faz mais do que apenas clonar o repositório (hospedado no Github com a organização ``the-fast-track``: ``https://github.com/the-fast-track/book-5.0-6``). Ele também inicia o servidor web, os containers, migra o banco de dados, carrega fixtures, ... Depois de executar o comando, o site deve estar pronto para ser usado.

É 100% garantido que o código esteja sincronizado com o código do livro (use a URL exata do repositório listado acima). Tentar sincronizar manualmente as alterações do livro com o código-fonte no repositório é quase impossível. Eu tentei no passado. E falhei. É simplesmente impossível. Especialmente para livros como os que eu escrevo: livros que contam uma história sobre o desenvolvimento de um site. Como cada capítulo depende dos anteriores, uma mudança pode ter consequências em todos os capítulos seguintes.

A boa notícia é que o repositório Git para este livro é *gerado automaticamente* a partir do conteúdo do livro. Você leu isso direito. Eu gosto de automatizar tudo, então há um script cujo trabalho é ler o livro e criar o repositório Git. Há um bom efeito colateral: ao atualizar o livro, o script falhará se as alterações forem inconsistentes ou se eu esquecer de atualizar algumas instruções. Isso é BDD, Book Driven Development!

Navegando no Código-Fonte
--------------------------

Melhor ainda, o repositório não é apenas sobre a versão final do código no branch ``master``. O script executa cada ação explicada no livro e faz o commit do seu trabalho no final de cada seção. Ele também cria uma tag a cada passo e subpasso para facilitar a navegação no código. Bonito, não é?

.. index::
    single: Git;checkout

Se você é preguiçoso, você pode obter o estado do código no final de uma etapa fazendo o checkout da tag relacionada. Por exemplo, se você gostaria de ler e testar o código no final do passo 10, execute o seguinte:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10

Assim como ao clonar o repositório, nós não estamos usando ``git checkout``, mas ``symfony book:checkout``. O comando assegura que qualquer que seja o estado em que você está atualmente, você acabará com um site funcional para o passo que você pedir. **Esteja ciente de que todos os dados, código e containers são removidos por esta operação.**

Você também pode fazer o checkout de qualquer subetapa:

.. code-block:: bash
    :class: ignore

    $ symfony book:checkout 10.2

De novo, recomendo fortemente que você mesmo escreva os códigos. Mas se você travar, você pode sempre comparar o que você fez com o conteúdo do livro.

.. index::
    single: Git;diff

Não tem certeza se fez tudo certo na subetapa 10.2? Compare os códigos:

.. code-block:: bash
    :class: ignore

    $ git diff step-10-1...step-10-2

    # And for the very first substep of a step:
    $ git diff step-9...step-10-1

.. index::
    single: Git;log

Quer saber quando um arquivo foi criado ou modificado?

.. code-block:: bash
    :class: ignore

    $ git log -- src/Controller/ConferenceController.php

Você também pode comparar códigos, tags e commits diretamente no GitHub. Esta é uma ótima maneira de copiar/colar código se você estiver lendo o livro em papel!
