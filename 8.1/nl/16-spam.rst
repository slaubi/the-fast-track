Spam voorkomen middels een API
==============================

.. index::
    single: Spam

Iedereen kan feedback geven. Ook scripts zoals robots en spammers. We zouden een "captcha" kunnen toevoegen aan het formulier waardoor het een zeker niveau van bescherming krijgt tegen dit soort scripts. Of, we kunnen gebruik maken van een aantal externe API's.

Ik heb besloten om de gratis `Akismet`_ service te gebruiken, om te laten zien hoe je een API kunt aanroepen en hoe je de calls "out of band" kunt maken.

Aanmelden bij Akismet
---------------------

.. index::
    single: Akismet

Meld je aan voor een gratis account op `akismet.com`_ en ontvang de Akismet API key.

Gebruik maken van het Symfony HTTPClient-component
--------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

In plaats van een specifieke library te gebruiken voor de Akismet API, zullen we alle API-calls direct uitvoeren. Het zelf uitvoeren van de HTTP-calls is efficiënter (en stelt ons in staat om te profiteren van alle Symfony debugging tools, zoals de integratie met de Symfony Profiler).

Een spam-checker-class bouwen
-----------------------------

Creëer een nieuwe class onder de ``src/`` map met de naam ``SpamChecker``. Hierin zullen we de logica schrijven om de Akismet API aan te roepen en de antwoorden te verwerken:

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

De ``request()`` methode van de HTTP client verstuurt een POST request naar de Akismet URL ( ``$this->endpoint`` ) en geeft hier een reeks parameters aan mee.

De ``getSpamScore()`` method geeft, afhankelijk van de API-response één van deze 3 waarden terug:

* ``2``: als de reactie "overduidelijk spam" is;

* ``1``: als de reactie "mogelijk spam" kan zijn;

* ``0``: als de reactie "geen spam" (ham) is.

.. tip::

    Gebruik het speciale ``akismet-guaranteed-spam@example.com`` e-mailadres om het resultaat van een call als spam te forceren.

Omgevingsvariabelen gebruiken
-----------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

De ``SpamChecker`` class is afhankelijk van een ``$akismetKey`` argument. Net als bij de upload directory, kunnen we deze injecteren via een ``Autowire`` annotatie:

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

We willen de waarde van de Akismet key niet hardcoded in de code. In plaats daarvan gebruiken we de omgevingsvariabele ( ``AKISMET_KEY`` ).

Het is dan aan de individuele ontwikkelaar om een "echte" omgevingsvariabele in te stellen of de waarde op te slaan in een ``.env.local`` bestand:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Voor productie moet een "echte" environment variable gedefiniëerd worden.

Dat werkt goed, maar het beheer van veel environment variables kan omslachtig worden. Symfony heeft een "beter" alternatief als het gaat om het bewaren van secrets.

Secrets bewaren
---------------

.. index::
    single: Secret

In plaats van veel omgevingsvariabelen te gebruiken, kan Symfony een *vault* beheren waar je secrets kan opslaan. Een belangrijke feature hiervan is de mogelijkheid om de vault aan de repository toe te voegen (maar zonder de decryptiesleutel om de inhoud te lezen). Een andere interessante feature is dat je één vault per omgeving kan beheren.

.. index:: ! Command;secrets:set

Secrets zijn verkapte environment variables.

Voeg de Akismet key toe aan de vault:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Omdat dit de eerste keer is dat we het commando uitvoeren zijn er twee keys in de ``config/secret/dev/`` map gegenereerd. De ``AKISMET_KEY`` secret werd vervolgens in diezelfde map opgeslagen.

Voor development-secrets kan je er voor kiezen om de vault en de sleutels die in de ``config/secret/dev/`` directory zijn gegenereerd te committen.

Secrets kunnen ook worden overschreven door een environment variable met dezelfde naam in te stellen.

Reacties controleren op spam
----------------------------

Een eenvoudige manier om nieuwe reacties te controleren op spam, is de spam checker aanroepen voordat de gegevens in de database worden opgeslagen:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
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

Controleer of het goed werkt.

Secrets beheren in productie
----------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

In productie ondersteunt Upsun het instellen van *sensitive environment variables*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

Zoals hierboven besproken is het gebruik van Symfony secrets mogelijk beter. Niet voor de veiligheid, maar om het beheer van secrets eenvoudiger te maken voor het projectteam. Alle secrets worden opgeslagen in de repository en de enige omgevingsvariabele die je moet beheren voor de productieomgeving is de decryptiesleutel. Dat maakt het voor iedereen in het team mogelijk om productie-secrets toe te voegen, zelfs als ze geen toegang hebben tot productieservers. De setup is wel iets complexer.

.. index::
    single: Command;secrets:generate-keys

Genereer eerst een keypair voor gebruik in productie:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Voeg het Akismet secret opnieuw toe in de productie vault, maar nu met de productie waarde:

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

Als laatste stap configureren we de decryptiesleutel op Upsun door het instellen van een sensitive variable:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Je kan alle bestanden toevoegen en committen; de decryptionkey werd automatisch aan ``.gitignore`` toegevoegd, dus deze zal nooit gecommit worden. Voor meer veiligheid kan je deze van je lokale machine verwijderen, omdat deze al gedeployd is:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Verder gaan

    * De `HttpClient-component documentatie`_;

    * De `Environment Variable Processors`_;

    * De `Symfony HttpClient Cheat Sheet`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`HttpClient-component documentatie`: https://symfony.com/doc/current/components/http_client.html
.. _`Environment Variable Processors`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Symfony HttpClient Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
