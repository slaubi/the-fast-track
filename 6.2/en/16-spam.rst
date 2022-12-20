Preventing Spam with an API
===========================

.. index::
    single: Spam

Anyone can submit a feedback. Even robots, spammers, and more. We could add some "captcha" to the form to somehow be protected from robots, or we can use some third-party APIs.

I have decided to use the free `Akismet`_ service to demonstrate how to call an API and how to make the call "out of band".

Signing up on Akismet
---------------------

.. index::
    single: Akismet

Sign-up for a free account on `akismet.com`_ and get the Akismet API key.

Depending on Symfony HTTPClient Component
-----------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Instead of using a library that abstracts the Akismet API, we will do all the API calls directly. Doing the HTTP calls ourselves is more efficient (and allows us to benefit from all the Symfony debugging tools like the integration with the Symfony Profiler).

Designing a Spam Checker Class
------------------------------

Create a new class under ``src/`` named ``SpamChecker`` to wrap the logic of calling the Akismet API and interpreting its responses:

.. code-block:: php
    :emphasize-lines: 14,24
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\Contracts\HttpClient\HttpClientInterface;

    class SpamChecker
    {
        private $endpoint;

        public function __construct(
            private HttpClientInterface $client,
            string $akismetKey,
        ) {
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

The HTTP client ``request()`` method submits a POST request to the Akismet URL (``$this->endpoint``) and passes an array of parameters.

The ``getSpamScore()`` method returns 3 values depending on the API call response:

* ``2``: if the comment is a "blatant spam";

* ``1``: if the comment might be spam;

* ``0``: if the comment is not spam (ham).

.. tip::

    Use the special ``akismet-guaranteed-spam@example.com`` email address to force the result of the call to be spam.

Using Environment Variables
---------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

The ``SpamChecker`` class relies on an ``$akismetKey`` argument. Like for the upload directory, we can inject it via an ``Autowire`` annotation:

.. code-block:: diff
    :caption: patch_file

    --- a/src/SpamChecker.php
    +++ b/src/SpamChecker.php
    @@ -3,6 +3,7 @@
     namespace App;

     use App\Entity\Comment;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Contracts\HttpClient\HttpClientInterface;

     class SpamChecker
    @@ -11,7 +12,7 @@ class SpamChecker

         public function __construct(
             private HttpClientInterface $client,
    -        string $akismetKey,
    +        #[Autowire('%env(AKISMET_KEY)%')] string $akismetKey,
         ) {
             $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
         }

We certainly don't want to hard-code the value of the Akismet key in the code, so we are using an environment variable instead (``AKISMET_KEY``).

It is then up to each developer to set a "real" environment variable or to store the value in a ``.env.local`` file:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

For production, a "real" environment variable should be defined.

That works well, but managing many environment variables might become cumbersome. In such a case, Symfony has a "better" alternative when it comes to storing secrets.

Storing Secrets
---------------

.. index::
    single: Secret

Instead of using many environment variables, Symfony can manage a *vault* where you can store many secrets. One key feature is the ability to commit the vault in the repository (but without the key to open it). Another great feature is that it can manage one vault per environment.

.. index:: ! Command;secrets:set

Secrets are environment variables in disguise.

Add the Akismet key in the vault:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Because this is the first time we have run this command, it generated two keys into the ``config/secret/dev/`` directory. It then stored the ``AKISMET_KEY`` secret in that same directory.

For development secrets, you can decide to commit the vault and the keys that have been generated in the ``config/secret/dev/`` directory.

Secrets can also be overridden by setting an environment variable of the same name.

Checking Comments for Spam
--------------------------

One simple way to check for spam when a new comment is submitted is to call the spam checker before storing the data into the database:

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
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -35,6 +36,7 @@ class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
         ): Response {
             $comment = new Comment();
    @@ -53,6 +55,17 @@ class ConferenceController extends AbstractController
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

Managing Secrets in Production
------------------------------

.. index::
    single: Platform.sh;Secret
    single: Platform.sh;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

For production, Platform.sh supports setting *sensitive environment variables*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

But as discussed above, using Symfony secrets might be better. Not in terms of security, but in terms of secret management for the project's team. All secrets are stored in the repository and the only environment variable you need to manage for production is the decryption key. That makes it possible for anyone in the team to add production secrets even if they don't have access to production servers. The setup is a bit more involved though.

.. index::
    single: Command;secrets:generate-keys

First, generate a pair of keys for production use:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Re-add the Akismet secret in the production vault but with its production value:

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

The last step is to send the decryption key to Platform.sh by setting a sensitive variable:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

You can add and commit all files; the decryption key has been added in ``.gitignore`` automatically, so it will never be committed. For more safety, you can remove it from your local machine as it has been deployed now:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Going Further

    * The `HttpClient component docs`_;

    * The `Environment Variable Processors`_;

    * The `Symfony HttpClient Cheat Sheet`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`HttpClient component docs`: https://symfony.com/doc/current/components/http_client.html
.. _`Environment Variable Processors`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Symfony HttpClient Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
