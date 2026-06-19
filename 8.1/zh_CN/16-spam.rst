用 API 防止垃圾信息
==========================

.. index::
    single: Spam

任何人都可以提交反馈。甚至网上的自动程序、垃圾信息发送器或其它程序都可以。我们可以在表单中加上某种“视觉验证码”来阻止自动程序，或者也可以用某些第三方提供的 API。

我决定使用免费的 `Akismet <https://akismet.com>`_ 服务，用它来演示如何调用 API，如何向“外部世界”发出请求。

在 Akismet 上注册
---------------------

.. index::
    single: Akismet

在 `akismet.com <https://akismet.com>`_ 上注册一个免费账号，然后获取 Akismet 的 API 秘钥。

依赖 Symfony 的 HTTPClient 组件
------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

我们会直接调用 Akismet 的 API，而不是使用一个包装了它 API 的库。我们自己发送 HTTP 请求会更加高效（因为集成了 Symfony 分析器组件，我们可以从 Symfony 调试工具中查看 API 调用情况）。

用 Symfony 的 HttpClient 组件来发起 API 调用：

.. code-block:: terminal

    $ symfony composer req http-client

设计一个垃圾信息检查器的类
---------------------------------------

在 ``src/`` 目录下新建一个名为 ``SpamChecker`` 的类，它会把调用Akismet API 和解析返回值的逻辑包装起来。

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

HTTP 客户端类的 ``request()`` 方法提交一个 POST 请求到 Akismet 的 URL（``$this->endpoint``），并且传递一组参数。

``getSpamScore()`` 方法根据 API 调用的结果返回 3 种值：

* ``2``：如果评论是一条“确凿无疑的垃圾信息”；

* ``1``：如果评论可能会是一条垃圾信息；

* ``0``：如果评论不是一条垃圾信息（就是所谓的 ham）。

.. tip::

    使用 ``akismet-guaranteed-spam@example.com`` 这个特别的邮箱地址作为参数，这样会强制让 API 调用判断为垃圾信息。

使用环境变量
------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``SpamChecker`` 类依赖于 ``$akismetKey`` 参数。和上传目录参数的处理方式一样，你可以通过把它“绑定”到容器的配置中来注入它：

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

我们当然不想把 Akismet 秘钥的值硬编码到 ``services.yaml`` 配置文件，所以我们用环境变量来代替（``AKISMET_KEY``）。

每个开发者可以自行决定，是设置一个“真正的”环境变量，还是把它的值存储在 ``.env.local`` 文件中：

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

在生产环境中，应该定义一个“真正的”环境变量。

这个方式很好，但管理许多环境变量会变得很麻烦。在这种情况下，Symfony 有一个“更好”的替代方案来存储机密信息。

存储机密信息
------------------

.. index::
    single: Secret

Symfony 可以管理用来存储许多机密信息的 *保险箱*，这可以取代使用很多环境变量。它的一个重要功能是可以把保险箱提交到代码仓库（但是不提交用来解密的 Key 文件）。另一个很棒的功能是每个环境可以有自己的保险箱。

.. index:: ! Command;secrets:set

机密信息就是伪装的环境变量。

把 Akismet 秘钥装进保险箱：

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

因为这是我们第一次运行这个命令，它会在 ``config/secret/dev/`` 目录下生成两个秘钥。然后它会把 ``AKISMET_KEY`` 这个机密信息也存放在同样的目录下。

对于开发环境下的机密信息，你可以决定来把这个保险箱以及 ``config/secret/dev/`` 目录下生成的秘钥提交到代码仓库。

可以通过设置同名的环境变量来覆盖机密信息的值。

测试评论是否为垃圾信息
---------------------------------

当有新评论提交时，需要对它进行检查，最好的时机就是在把它存储到数据库之前，调用垃圾信息检查器做这个检查。

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

检查程序是否正常工作。

在生产环境中管理机密信息
------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

在生产环境中，Upsun 支持设置 *敏感环境变量*：

.. code-block:: terminal
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

正如以上讨论的，使用 Symfony 的机密信息管理会更好。这并不是从安全性上考虑，而是从项目团队对于机密信息的管理上来考虑。所有的机密信息存放在代码仓库中，你唯一需要管理的生产环境所需的环境变量就是解密秘钥。这样团队中的任何人都可以增加生产环境中的机密信息，即便他们无权访问生产服务器。不过对此的设置会更复杂些。

.. index::
    single: Command;secrets:generate-keys

首先，为生产环境生成一对秘钥：

.. code-block:: terminal

    $ APP_ENV=prod symfony console secrets:generate-keys

.. note::

    The ``APP_ENV=prod`` part before the command allows setting the ``APP_ENV`` environment variable only for this command. On Windows, use ``--env=prod`` instead: ``symfony console secrets:generate-keys --env=prod``

.. index::
    single: Command;secrets:set

把 Akismet 的秘钥加入生产环境的保险箱中，但这次是用针对生产环境的值：

.. code-block:: terminal
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

最后一步是通过设置一个敏感环境变量，把解密秘钥发送给 Upsun：

.. code-block:: terminal

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

你可以添加和提交所有文件；解密秘钥已经被自动加到 ``.gitignore`` 里了，所以它永远不会被提交。既然已经部署完了，出于安全考虑，你可以将它从你的本地电脑中移除：

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: 深入学习

    * `HttpClient 组件文档 <https://symfony.com/doc/current/components/http_client.html>`_；

    * `环境变量处理器 <https://symfony.com/doc/current/configuration/env_var_processors.html>`_；

    * `Symfony HttpClient 速查表 <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_。
