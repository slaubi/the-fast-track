Preventing Spam with AI
=======================

.. index::
    single: Spam

Anyone can submit a feedback. Even robots, spammers, and more. We could add some "captcha" to the form to somehow be protected from robots, or we can use some third-party APIs.

I have decided to use a Large Language Model to decide whether a comment is spam, to demonstrate how to use AI in a Symfony application and how to make such expensive calls "out of band".

Getting an AI API Key
---------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI supports many model providers: OpenAI, Anthropic, Google Gemini, Mistral, and even local models via Ollama. This chapter uses OpenAI: sign-up on `platform.openai.com`_ and create an API key. If you prefer another provider, the code stays the same; only the configuration changes.

Depending on the Symfony AI Bundle
----------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Instead of calling the model's HTTP API ourselves, we will use the Symfony AI Bundle. It provides a *platform* abstraction for the model providers (each provider comes as its own bridge package) and an *agent* that wraps a model to make calls; and it benefits from all the Symfony debugging tools like the integration with the Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI is a young set of components and still experimental: its APIs may evolve faster than the rest of Symfony.

The OpenAI bridge recipe has already configured the platform for us; it references an ``OPENAI_API_KEY`` environment variable (and added an empty default for it in ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Configure a default *agent* on top of it:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Using Environment Variables
---------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

We certainly don't want to hard-code the key's value in the configuration; that's why it is read from the ``OPENAI_API_KEY`` environment variable.

It is then up to each developer to set a "real" environment variable or to store the value in a ``.env.local`` file:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

For production, a "real" environment variable should be defined.

That works well, but managing many environment variables might become cumbersome. In such a case, Symfony has a "better" alternative when it comes to storing secrets.

Storing Secrets
---------------

.. index::
    single: Secret

Instead of using many environment variables, Symfony can manage a *vault* where you can store many secrets. One key feature is the ability to commit the vault to the repository (but without the key to open it). Another great feature is that it can manage one vault per environment.

.. index:: ! Command;secrets:set

Secrets are environment variables in disguise.

Add the OpenAI API key in the vault:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Because this is the first time we have run this command, it generated two keys into the ``config/secret/dev/`` directory. It then stored the ``OPENAI_API_KEY`` secret in that same directory.

For development secrets, you can decide to commit the vault and the keys that have been generated in the ``config/secret/dev/`` directory.

Secrets can also be overridden by setting an environment variable of the same name.

.. index::
    single: Command;secrets:reveal

To read a secret back from the vault, use ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Designing a Spam Checker Class
------------------------------

.. index::
    single: AI;Prompt

Create a new class under ``src/`` named ``SpamChecker`` to wrap the logic of asking the model whether a comment is spam:

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

The *system prompt* tells the model its role and constrains its answers; the *user message* contains the comment and its submission context (IP address, user agent).

The ``getSpamScore()`` method returns 3 values depending on the model's answer:

* ``2``: if the comment is a "blatant spam";

* ``1``: if the comment might be spam, or when the model cannot be reached;

* ``0``: if the comment is not spam (ham).

A model's output is free text, even when the prompt constrains it: parse it liberally (lowercase it, use ``str_contains()``). And when the model cannot answer at all, fall back to human moderation instead of failing: AI should help the admin, never block the guestbook.

.. tip::

    Try submitting a comment that looks blatantly spammy, like "Buy cheap watches at http://example.com/!!!", to see the model at work.

Checking Comments for Spam
--------------------------

One simple way to check for spam when a new comment is submitted is to call the spam checker before storing the data into the database:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,7 +35,8 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
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

Check that it works fine.

Rate Limiting Comment Submissions
---------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Detecting spam protects the website against sophisticated spammers. A complementary and much cheaper protection is to limit how fast the same client can submit comments: nobody legitimately posts dozens of comments per hour on a guestbook.

Add the Symfony Rate Limiter component:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Configure a limiter that accepts at most 5 comments per hour from the same client:

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

Automated tests legitimately submit many comments in a short period of time, so the limit is raised for the ``test`` environment.

Enforce the limiter on comment submissions with the ``#[RateLimit]`` attribute; by default, it identifies clients by their IP address:

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
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,

Note the ``methods`` argument: browsing a conference page is a ``GET`` request and must not be limited; only comment submissions (``POST`` requests) are.

When the limit is reached, Symfony automatically returns a ``429 Too Many Requests`` response with a ``Retry-After`` HTTP header telling the client when it can retry.

The same component also protects the admin login form against brute-force attacks; enabling *login throttling* on the firewall takes one line:

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

By default, Symfony blocks an IP after 5 failed login attempts on the same username within a minute (a successful login resets the counter). Use the ``max_attempts`` and ``interval`` options to tune the policy.

Managing Secrets in Production
------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

For production, Upsun supports setting *sensitive environment variables*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

But as discussed above, using Symfony secrets might be better. Not in terms of security, but in terms of secret management for the project's team. All secrets are stored in the repository and the only environment variable you need to manage for production is the decryption key. That makes it possible for anyone in the team to add production secrets even if they don't have access to production servers. The setup is a bit more involved though.

.. index::
    single: Command;secrets:generate-keys

First, generate a pair of keys for production use:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    On Linux and similar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Re-add the OpenAI API key secret in the production vault but with its production value:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

The last step is to send the decryption key to Upsun by setting a sensitive variable:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

You can add and commit all files; the decryption key has been added in ``.gitignore`` automatically, so it will never be committed. For more safety, you can remove it from your local machine as it has been deployed now:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Going Further

    * The `Symfony AI documentation`_;

    * The `Environment Variable Processors`_;

    * `How to Keep Sensitive Information Secret`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Symfony AI documentation`: https://symfony.com/doc/current/ai/index.html
.. _`Environment Variable Processors`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`How to Keep Sensitive Information Secret`: https://symfony.com/doc/current/configuration/secrets.html
