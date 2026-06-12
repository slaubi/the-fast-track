AI でスパム対策をする
=====================

.. index::
    single: Spam

ロボットやスパマーなど誰でもフィードバックを投稿すること可能な状態ですので、 "CAPTCHA" を追加したり、サードパーティのAPI を使用して、ロボットからの投稿から保護することを考えます。

ここでは、コメントがスパムかどうかの判定を大規模言語モデル（LLM）に任せることにしました。Symfony アプリケーションでの AI の使い方と、こうしたコストの高い呼び出しをアウトオブバンドで行う方法を説明します。

AI の API キーを取得する
------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI は、OpenAI、Anthropic、Google Gemini、Mistral、さらには Ollama 経由のローカルモデルなど、多くのモデルプロバイダーをサポートしています。この章では OpenAI を使用します。 `platform.openai.com`_ に登録し、API キーを作成してください。他のプロバイダーを使用したい場合でも、コードは同じで、変わるのは設定だけです。

Symfony AI バンドルに依存させる
-------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

モデルの HTTP API を直接呼ぶのではなく、Symfony AI バンドルを使用します。このバンドルは、モデルプロバイダーの *プラットフォーム* 抽象化（各プロバイダーは専用のブリッジパッケージとして提供されます）と、モデルをラップして呼び出しを行う *エージェント* を提供します。さらに、Symfony プロファイラとの統合など、Symfony のデバッグツールの恩恵も得られます:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI はまだ新しい実験的なコンポーネント群です。API は Symfony 本体よりも速いペースで変わる可能性があります。

OpenAI ブリッジのレシピが、既にプラットフォームを設定してくれています。 ``OPENAI_API_KEY`` 環境変数を参照しています（ ``.env`` に空のデフォルト値も追加されています）:

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

その上にデフォルトの *エージェント* を設定してください:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

環境変数を使用する
------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

コードにハードコードでキーの値を書くことは避けたいですので、 ``OPENAI_API_KEY`` 環境変数から読み込むようになっています。

"本当の" 環境変数をセットするか ``.env.local`` ファイルに値をセットするかは各エンジニアの判断に任せます:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

本番においては、 "本当の" 環境変数を定義するべきです。

多くの環境変数を管理するのは大変ですので、 Symfony は、シークレット情報を格納するのに "ベター" な方法があります。

シークレット情報を格納する
--------------------------------

.. index::
    single: Secret

たくさんの環境変数を使用する代わりに、Symfony では、シークレット情報を格納することができる *ヴォールト* で管理することができます。例えば、メリットの一つとしてリポジトリにヴォールトをコミットすることができます（キーは入れないでください）。さらに、環境毎にヴォールトを管理することも可能です。

.. index:: ! Command;secrets:set

シークレット情報は、実際の値を覆った環境変数になります。

OpenAI API キーをヴォールトに追加します:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

このコマンドを実行するのは初めてなので、 ``config/secret/dev`` ディレクトリにキーが2つ生成されます。そして、 ``OPENAI_API_KEY`` シークレットが同ディレクトリに格納されます。

開発時のシークレットでは、 ``config/secret/dev`` ディレクトリに生成されたヴォールトとそのキーをコミットすることもできます。

同名の環境変数をセットすることでシークレットは上書きすることも可能です。

.. index::
    single: Command;secrets:reveal

ヴォールトからシークレットを読み出すには、 ``secrets:reveal`` を使用してください:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

スパムチェッカークラスを設計する
------------------------------------------------

.. index::
    single: AI;Prompt

``src/`` 以下に、新しいクラス ``SpamChecker`` を追加し、コメントがスパムかどうかをモデルに尋ねるロジックをラップさせます:

.. code-block:: php
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\AI\Agent\AgentInterface;
    use Symfony\AI\Platform\Exception\ExceptionInterface;
    use Symfony\AI\Platform\Message\Message;
    use Symfony\AI\Platform\Message\MessageBag;

    class SpamChecker
    {
        public function __construct(
            private AgentInterface $agent,
        ) {
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $messages = new MessageBag(
                Message::forSystem(<<<PROMPT
                    You moderate comments submitted to a conference guestbook.
                    Classify the comment as "ham", "maybe spam", or "blatant spam".
                    Only answer with the classification.
                    PROMPT),
                Message::ofUser(sprintf(<<<COMMENT
                    IP: %s
                    User agent: %s
                    Author: %s (%s)
                    Comment: %s
                    COMMENT,
                    $context['user_ip'] ?? '',
                    $context['user_agent'] ?? '',
                    $comment->getAuthor(),
                    $comment->getEmail(),
                    $comment->getText(),
                )),
            );

            try {
                $answer = strtolower($this->agent->call($messages)->getContent());
            } catch (ExceptionInterface) {
                // when the model cannot answer, let a human moderate the comment
                return 1;
            }

            return match (true) {
                str_contains($answer, 'blatant spam') => 2,
                str_contains($answer, 'maybe spam') => 1,
                default => 0,
            };
        }
    }

*システムプロンプト* は、モデルに役割を伝え、回答を制約します。 *ユーザーメッセージ* には、コメントと投稿のコンテキスト（IP アドレス、ユーザーエージェント）が含まれています。

``getSpamScore()`` メソッドは、モデルの回答に応じて 3 つの値を返します:

* ``2``: コメントが明らかなスパム（"blatant spam"）の場合;

* ``1``: コメントがスパムの可能性がある場合、またはモデルに到達できない場合;

* ``0``: コメントがスパムでない場合（ham）。

モデルの出力は、プロンプトで制約していても自由なテキストです。寛容にパースしてください（小文字に変換して ``str_contains()`` を使用します）。そして、モデルがまったく回答できないときは、失敗させるのではなく人間によるモデレーションにフォールバックしてください。AI は管理者を助けるべきであり、ゲストブックをブロックしてはいけません。

.. tip::

    "Buy cheap watches at http://example.com/!!!" のような明らかにスパムに見えるコメントを投稿して、モデルの動作を確認してみてください。

コメントがスパムかチェックする
---------------------------------------------

新しいコメントが投稿されたときにスパムかチェックする簡単な方法の一つとして、データベースに保存する前にスパムチェッカーを呼び出すことです:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,7 +7,8 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,8 +35,9 @@ final class ConferenceController extends AbstractController
             Request $request,
             #[MapEntity(mapping: ['slug' => 'slug'])]
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -48,6 +50,17 @@ final class ConferenceController extends AbstractController
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

コメント投稿のレートを制限する
------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

スパム検知は、巧妙なスパマーから Web サイトを守ります。それを補完する、はるかに安価な保護が、同じクライアントがコメントを投稿できる頻度を制限することです。1 時間に何十件もコメントを投稿する正当なユーザーはいません。

Symfony Rate Limiter コンポーネントを追加してください:

.. code-block:: terminal

    $ symfony composer req rate-limiter

同じクライアントからのコメントを 1 時間に最大 5 件まで受け付けるリミッターを設定します:

.. code-block:: yaml
    :caption: config/packages/rate_limiter.yaml

    framework:
        rate_limiter:
            comment_submission:
                policy: 'fixed_window'
                limit: 5
                interval: '1 hour'

    when@test:
        framework:
            rate_limiter:
                comment_submission:
                    limit: 1000

自動テストは正当な理由で短時間にたくさんのコメントを投稿するので、 ``test`` 環境では制限を引き上げています。

``#[RateLimit]`` アトリビュートでコメント投稿にリミッターを適用してください。デフォルトでは、クライアントは IP アドレスで識別されます:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    +use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -31,6 +32,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,

``methods`` 引数に注目してください。カンファレンスページの閲覧は ``GET`` リクエストであり、制限してはいけません。制限されるのはコメント投稿（ ``POST`` リクエスト）だけです。

制限に達すると、Symfony は自動的に ``429 Too Many Requests`` レスポンスを返します。 ``Retry-After`` HTTP ヘッダーで、クライアントがいつ再試行できるかを伝えます。

同じコンポーネントは、ブルートフォース攻撃から管理者のログインフォームも守ってくれます。ファイアウォールで *ログインスロットリング* を有効にするのは 1 行で済みます:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -19,6 +19,7 @@ security:
             main:
                 lazy: true
                 provider: app_user_provider
    +            login_throttling: ~
                 form_login:
                     login_path: app_login
                     check_path: app_login

デフォルトでは、Symfony は同じユーザー名に対して 1 分間に 5 回ログインに失敗した IP をブロックします（ログインに成功するとカウンターはリセットされます）。ポリシーを調整するには ``max_attempts`` と ``interval`` オプションを使用してください。

本番でシークレットを管理する
------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

本番では、 Upsun は *注意が必要な環境変数* の設定をサポートしています:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

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

本番用の OpenAI API キーのシークレットを、本番の値で本番のヴォールトに再追加してください:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

最後に、Upsun に、注意が必要な値をセットした際の復号キーを送ってください:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

復号キーは、 ``.gitignore`` に自動的に追加されているので、コミットされることはありませんので、全てのファイルを追加することができます。デプロイが終わったので、安全のために、ローカルマシンから削除しておいてください:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: より深く学ぶために

    * `Symfony AI のドキュメント`_;

    * `環境変数プロセッサー`_;

    * `機密情報を秘密にしておく方法`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Symfony AI のドキュメント`: https://symfony.com/doc/current/ai/index.html
.. _`環境変数プロセッサー`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`機密情報を秘密にしておく方法`: https://symfony.com/doc/current/configuration/secrets.html
