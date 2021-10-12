Prevenindo Spam com uma API
===========================

.. index::
    single: Spam

Qualquer um pode enviar um feedback. Até robôs, spammers e muitos outros. Poderíamos adicionar algum "captcha" ao formulário para, de alguma forma, nos proteger de robôs, ou podemos usar algumas APIs de terceiros.

Decidi usar o serviço gratuito `Akismet <https://akismet.com>`_ para demonstrar como chamar uma API e como fazer a chamada "em outro momento".

Fazendo o Cadastro no Akismet
-----------------------------

.. index::
    single: Akismet

Cadastre-se para uma conta gratuita no `akismet.com <https://akismet.com>`_ e obtenha a chave da API do Akismet.

Dependendo do Componente HTTPClient do Symfony
----------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Em vez de usar uma biblioteca que abstrai a API do Akismet, faremos todas as chamadas de API diretamente. Fazer as chamadas HTTP diretamente é mais eficiente (e nos permite nos beneficiar de todas as ferramentas de depuração do Symfony, como a integração com o Profiler).

Para fazer chamadas de API, use o Componente HttpClient do Symfony:

.. code-block:: bash

    $ symfony composer req http-client

Projetando uma Classe Verificadora de Spam
------------------------------------------

Crie uma nova classe em ``src/`` chamada ``SpamChecker`` para encapsular a lógica de chamar a API do Akismet e interpretar suas respostas:

.. code-block:: php
    :emphasize-lines: 14,24
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\Contracts\HttpClient\HttpClientInterface;

    class SpamChecker
    {
        private $client;
        private $endpoint;

        public function __construct(HttpClientInterface $client, string $akismetKey)
        {
            $this->client = $client;
            $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         *
         * @throws \RuntimeException if the call did not work
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $response = $this->client->request('POST', $this->endpoint, [
                'body' => array_merge($context, [
                    'blog' => 'https://guestbook.example.com',
                    'comment_type' => 'comment',
                    'comment_author' => $comment->getAuthor(),
                    'comment_author_email' => $comment->getEmail(),
                    'comment_content' => $comment->getText(),
                    'comment_date_gmt' => $comment->getCreatedAt()->format('c'),
                    'blog_lang' => 'en',
                    'blog_charset' => 'UTF-8',
                    'is_test' => true,
                ]),
            ]);

            $headers = $response->getHeaders();
            if ('discard' === ($headers['x-akismet-pro-tip'][0] ?? '')) {
                return 2;
            }

            $content = $response->getContent();
            if (isset($headers['x-akismet-debug-help'][0])) {
                throw new \RuntimeException(sprintf('Unable to check for spam: %s (%s).', $content, $headers['x-akismet-debug-help'][0]));
            }

            return 'true' === $content ? 1 : 0;
        }
    }

O método ``request()`` do cliente HTTP envia uma requisição POST para a URL do Akismet (``$this->endpoint``) e passa um array de parâmetros.

O método ``getSpamScore()`` retorna 3 valores dependendo da resposta da chamada à API:

* ``2``: se o comentário for "claramente um spam";

* ``1``: se o comentário pode ser um spam;

* ``0``: se o comentário não for um spam (ham).

.. tip::

    Use o endereço de e-mail especial ``akismet-guaranteed-spam@example.com`` para forçar o resultado da chamada a ser spam.

Usando Variáveis de Ambiente
-----------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

A classe ``SpamChecker`` depende de um argumento ``$akismetKey``. Assim como para o diretório de upload, podemos injetá-lo através de uma configuração ``bind`` no container:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -12,6 +12,7 @@ services:
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
             bind:
                 $photoDir: "%kernel.project_dir%/public/uploads/photos"
    +            $akismetKey: "%env(AKISMET_KEY)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

Nós certamente não queremos codificar o valor da chave do Akismet diretamente no arquivo de configuração ``services.yaml``, então estamos usando uma variável de ambiente (``AKISMET_KEY``).

Depois, cabe a cada desenvolvedor definir uma variável de ambiente "real" ou armazenar o valor em um arquivo ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Para produção, uma variável de ambiente "real" deve ser definida.

Isso funciona bem, mas gerenciar muitas variáveis de ambiente pode se tornar complicado. Nesse caso, o Symfony tem uma alternativa "melhor" quando se trata de armazenar segredos.

Armazenando Segredos
--------------------

.. index::
    single: Secret

Em vez de usar muitas variáveis de ambiente, o Symfony pode gerenciar um *cofre* onde você pode armazenar muitos segredos. Uma característica essencial é a habilidade de fazer o commit do cofre no repositório (mas sem a chave para abri-lo). Outra grande característica é que ele pode gerenciar um cofre por ambiente.

.. index:: ! Command;secrets:set

Os segredos são variáveis de ambiente disfarçadas.

Adicione a chave do Akismet no cofre:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Como esta é a primeira vez que executamos esse comando, ele gerou duas chaves no diretório ``config/secret/dev/``. Depois, armazenou o segredo ``AKISMET_KEY`` nesse mesmo diretório.

Para os segredos de desenvolvimento, você pode decidir fazer o commit do cofre e das chaves que foram geradas no diretório ``config/secret/dev/``.

Os segredos também podem ser substituídos definindo uma variável de ambiente com o mesmo nome.

Procurando Spam nos Comentários
--------------------------------

Uma maneira simples de verificar se há spam quando um novo comentário é enviado é chamar o verificador de spam antes de armazenar os dados no banco de dados:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\File\Exception\FileException;
    @@ -39,7 +40,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -57,6 +58,17 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +
    +            $context = [
    +                'user_ip' => $request->getClientIp(),
    +                'user_agent' => $request->headers->get('user-agent'),
    +                'referrer' => $request->headers->get('referer'),
    +                'permalink' => $request->getUri(),
    +            ];
    +            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    +                throw new \RuntimeException('Blatant spam, go away!');
    +            }
    +
                 $this->entityManager->flush();

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);

Verifique se funciona bem.

Gerenciando Segredos em Produção
----------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

Para produção, a SymfonyCloud suporta a configuração de *variáveis de ambiente sensíveis*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Mas, como discutimos antes, usar os segredos do Symfony pode ser melhor. Não em termos de segurança, mas em termos de gestão de segredos para o time do projeto. Todos os segredos são armazenados no repositório e a única variável de ambiente que você precisa gerenciar em produção é a chave de decodificação. Isso torna possível para qualquer um no time adicionar segredos de produção, mesmo que não tenha acesso a servidores de produção. A configuração é um pouco mais complicada.

.. index::
    single: Command;secrets:generate-keys

Primeiro, gere um par de chaves para uso em produção:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Adicione novamente o segredo do Akismet no cofre de produção, mas com o seu valor de produção:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

O último passo é enviar a chave de decodificação para a SymfonyCloud definindo uma variável sensível:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Você pode adicionar e fazer o commit de todos os arquivos; a chave de decodificação foi adicionada automaticamente ao ``.gitignore``, então ela nunca será incluída no commit. Para maior segurança, você pode removê-la da sua máquina local, já que agora ela foi implantada:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Indo Além

    * A `documentação do Componente HttpClient <https://symfony.com/doc/current/components/http_client.html>`_;

    * Os `Processadores de Variáveis de Ambiente <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * A `Cheat Sheet do HttpClient do Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
