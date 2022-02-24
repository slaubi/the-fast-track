API でスパム対策をする
===============================

.. index::
    single: Spam

ロボットやスパマーなど誰でもフィードバックを投稿すること可能な状態ですので、 "CAPTCHA" を追加したり、サードパーティのAPI を使用して、ロボットからの投稿から保護することを考えます。

`Akismet`_ を使用することにしましょう。ここでは、アウトオブバンドに Akismet の API を呼ぶ方法を説明します。

Akismet に登録する
-----------------------

.. index::
    single: Akismet

`akismet.com`_ の無料アカウントに登録し、Akismet APIキーを取得します。

Symfony HTTPClient コンポーネントに依存させる
----------------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Akismet API を抽象化したライブラリを使用するのではなく、まず直接API を呼んでみましょう。HTTP 呼び出しがより効率的です（Symfonyプロファイラが使用できるのでSymfonyデバッグツールの恩恵が得られます）。

スパムチェッカークラスを設計する
------------------------------------------------

``src/`` 以下に、新しいクラス ``SpamChecker`` を追加し、 Akismet API の呼び出しロジックをラップしてレスポンスを解釈させます:

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

HTTP クライアントの ``request()`` メソッドは、 Akismet URL(``$this->endpoint``) に POST リクエストを行い、パラメーターの配列を渡します。

``getSpamScore()`` メソッドは API 呼び出しのレスポンスに応じて 3つの値を返します:

* ``2``: コメントが "露骨なスパム";

* ``1`` コメントがスパムの可能性がある;

* ``0`` コメントがスパムでない。

.. tip::

    特別なメールアドレスの ``akismet-guaranteed-spam@example.com`` を使用すると強制的にスパムと判定させることができます。

環境変数を使用する
---------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``SpamChecker`` クラスは ``$akismetKey`` 引数が必要です。ファイルアップロードのディレクトリのときと同じようにコンテナの設定に ``バインド`` させてインジェクトします:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -12,6 +12,7 @@ services:
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
             bind:
                 string $photoDir: "%kernel.project_dir%/public/uploads/photos"
    +            string $akismetKey: "%env(AKISMET_KEY)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

``services.yaml`` にハードコードで Akismet のキーを書くことは避けたいですので、環境変数の (``AKISMET_KEY``) を使用することにします。

"本当の" 環境変数をセットするか ``.env.local`` ファイルに値をセットするかは各エンジニアの判断に任せます:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

本番においては、 "本当の" 環境変数を定義するべきです。

多くの環境変数を管理するのは大変ですので、 Symfony は、シークレット情報を格納するのに "ベター" な方法があります。

シークレット情報を格納する
---------------------------------------

.. index::
    single: Secret

たくさんの環境変数を使用する代わりに、Symfony では、シークレット情報を格納することができる *ヴォールト* で管理することができます。例えば、メリットの一つとしてリポジトリにヴォールトをコミットすることができます（キーは入れないでください）。さらに、環境毎にヴォールトを管理することも可能です。

.. index:: ! Command;secrets:set

シークレット情報は、実際の値を覆った環境変数になります。

Akismet Key をヴォールトに追加します:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

このコマンドを実行するのは初めてなので、 ``config/secret/dev`` ディレクトリにキーが2つ生成されます。そして、 ``AKISMET_KEY`` シークレットが同ディレクトリに格納されます。

開発時のシークレットでは、 ``config/secret/dev`` ディレクトリに生成されたヴォールトとそのキーをコミットすることもできます。

同名の環境変数をセットすることでシークレットは上書きすることも可能です。

コメントがスパムかチェックする
---------------------------------------------

新しいコメントが投稿されたときにスパムかチェックする簡単な方法の一つとして、データベースに保存する前にスパムチェッカーを呼び出すことです:

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
    @@ -35,7 +36,7 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -53,6 +54,17 @@ class ConferenceController extends AbstractController
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

正しく動作するかチェックする

本番でシークレットを管理する
------------------------------------------

.. index::
    single: Platform.sh;Secret
    single: Platform.sh;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

本番では、 Platform.sh は *注意が必要な環境変数* の設定をサポートしています:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

しかし、上記で議論したように、セキュリティーの面からではなくプロジェクトチームのシークレット管理の面から、Symfony のシークレット管理を使う方がベターです。全てのシークレットがリポジトリに格納されるので、唯一の管理するべき本番の環境変数は復号キーのみとなります。こうすることで、少しセットアップが面倒ですが、チームの誰もが本番のサーバーへのアクセス権がなくても、本番のシークレットを追加することができます。

.. index::
    single: Command;secrets:generate-keys

まず、本番用のキーのペアを生成してください:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

本番用の Akismet のシークレットを本番のヴォールトに再追加してください:

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

最後に、Platform.sh に、注意が必要な値をセットした際の復号キーを送ってください:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

復号キーは、 ``.gitignore`` に自動的に追加されているので、コミットされることはありませんので、全てのファイルを追加することができます。デプロイが終わったので、安全のために、ローカルマシンから削除しておいてください:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: より深く学ぶために

    * `HttpClient コンポーネントのドキュメント`_;

    * `環境変数プロセッサー`_;

    * `Symfony HttpClient チートシート`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`HttpClient コンポーネントのドキュメント`: https://symfony.com/doc/current/components/http_client.html
.. _`環境変数プロセッサー`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Symfony HttpClient チートシート`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
