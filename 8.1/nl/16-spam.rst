Spam voorkomen met AI
=====================

.. index::
    single: Spam

Iedereen kan feedback geven. Ook scripts zoals robots en spammers. We zouden een "captcha" kunnen toevoegen aan het formulier waardoor het een zeker niveau van bescherming krijgt tegen dit soort scripts. Of, we kunnen gebruik maken van een aantal externe API's.

Ik heb besloten om een Large Language Model te gebruiken om te bepalen of een reactie spam is, om te laten zien hoe je AI gebruikt in een Symfony-applicatie en hoe je zulke dure calls "out of band" maakt.

Een AI API-key verkrijgen
-------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI ondersteunt veel model-providers: OpenAI, Anthropic, Google Gemini, Mistral en zelfs lokale modellen via Ollama. Dit hoofdstuk gebruikt OpenAI: meld je aan op `platform.openai.com`_ en maak een API-key aan. Als je liever een andere provider gebruikt, blijft de code hetzelfde; alleen de configuratie verandert.

Afhankelijk zijn van de Symfony AI Bundle
-----------------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

In plaats van zelf de HTTP-API van het model aan te roepen, gebruiken we de Symfony AI Bundle. Deze biedt een *platform*-abstractie voor de model-providers (elke provider komt als zijn eigen bridge-package) en een *agent* die een model omhult om calls te maken; en het profiteert van alle Symfony debugging tools zoals de integratie met de Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI is een jonge set componenten en nog experimenteel: de API's kunnen sneller evolueren dan de rest van Symfony.

Het recept van de OpenAI-bridge heeft het platform al voor ons geconfigureerd; het verwijst naar een ``OPENAI_API_KEY`` omgevingsvariabele (en heeft er een lege standaardwaarde voor toegevoegd in ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Configureer er een standaard-*agent* bovenop:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Omgevingsvariabelen gebruiken
-----------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

We willen de waarde van de key zeker niet hardcoded in de configuratie zetten; daarom wordt deze gelezen uit de ``OPENAI_API_KEY`` omgevingsvariabele.

Het is dan aan de individuele ontwikkelaar om een "echte" omgevingsvariabele in te stellen of de waarde op te slaan in een ``.env.local`` bestand:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Voor productie moet een "echte" environment variable gedefiniëerd worden.

Dat werkt goed, maar het beheer van veel environment variables kan omslachtig worden. In dat geval heeft Symfony een "beter" alternatief als het gaat om het bewaren van secrets.

Secrets bewaren
---------------

.. index::
    single: Secret

In plaats van veel omgevingsvariabelen te gebruiken, kan Symfony een *vault* beheren waar je secrets kan opslaan. Een belangrijke feature hiervan is de mogelijkheid om de vault aan de repository toe te voegen (maar zonder de sleutel om de inhoud te openen). Een andere interessante feature is dat je één vault per omgeving kan beheren.

.. index:: ! Command;secrets:set

Secrets zijn verkapte environment variables.

Voeg de OpenAI API-key toe aan de vault:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Omdat dit de eerste keer is dat we het commando uitvoeren, zijn er twee keys in de ``config/secret/dev/`` map gegenereerd. De ``OPENAI_API_KEY`` secret werd vervolgens in diezelfde map opgeslagen.

Voor development-secrets kan je er voor kiezen om de vault en de sleutels die in de ``config/secret/dev/`` directory zijn gegenereerd te committen.

Secrets kunnen ook worden overschreven door een environment variable met dezelfde naam in te stellen.

.. index::
    single: Command;secrets:reveal

Om een secret terug te lezen uit de vault, gebruik je ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Een spam-checker-class bouwen
-----------------------------

.. index::
    single: AI;Prompt

Creëer een nieuwe class onder de ``src/`` map met de naam ``SpamChecker`` om de logica te omhullen die het model vraagt of een reactie spam is:

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

De *system prompt* vertelt het model zijn rol en beperkt zijn antwoorden; het *user message* bevat de reactie en de context van de inzending (IP-adres, user agent).

De ``getSpamScore()`` method geeft, afhankelijk van het antwoord van het model, één van deze 3 waarden terug:

* ``2``: als de reactie "overduidelijk spam" is;

* ``1``: als de reactie mogelijk spam is, of wanneer het model niet bereikbaar is;

* ``0``: als de reactie geen spam (ham) is.

De output van een model is vrije tekst, zelfs wanneer de prompt deze beperkt: parse het ruimhartig (zet het om naar kleine letters, gebruik ``str_contains()``). En wanneer het model helemaal geen antwoord kan geven, val dan terug op menselijke moderatie in plaats van te falen: AI moet de beheerder helpen, nooit het gastenboek blokkeren.

.. tip::

    Probeer een reactie in te dienen die er overduidelijk spammy uitziet, zoals "Buy cheap watches at http://example.com/!!!", om het model aan het werk te zien.

Reacties controleren op spam
----------------------------

Een eenvoudige manier om nieuwe reacties te controleren op spam, is de spam checker aanroepen voordat de gegevens in de database worden opgeslagen:

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

Controleer of het goed werkt.

Reactie-inzendingen beperken met rate limiting
----------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Spam detecteren beschermt de website tegen geraffineerde spammers. Een aanvullende en veel goedkopere bescherming is om te beperken hoe snel dezelfde client reacties kan insturen: niemand plaatst legitiem tientallen reacties per uur op een gastenboek.

Voeg het Symfony Rate Limiter-component toe:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Configureer een limiter die maximaal 5 reacties per uur van dezelfde client accepteert:

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

Geautomatiseerde tests dienen legitiem veel reacties in een korte tijd in, dus de limiet wordt verhoogd voor de ``test`` omgeving.

Pas de limiter toe op reactie-inzendingen met het ``#[RateLimit]`` attribuut; standaard identificeert het clients aan de hand van hun IP-adres:

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

Let op het ``methods`` argument: het bekijken van een conferentiepagina is een ``GET`` request en mag niet beperkt worden; alleen reactie-inzendingen (``POST`` requests) wel.

Wanneer de limiet is bereikt, geeft Symfony automatisch een ``429 Too Many Requests`` response terug met een ``Retry-After`` HTTP-header die de client vertelt wanneer het opnieuw kan proberen.

Hetzelfde component beschermt ook het admin-inlogformulier tegen brute-force aanvallen; het inschakelen van *login throttling* op de firewall kost één regel:

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

Standaard blokkeert Symfony een IP na 5 mislukte inlogpogingen op dezelfde gebruikersnaam binnen een minuut (een geslaagde login reset de teller). Gebruik de ``max_attempts`` en ``interval`` opties om het beleid af te stemmen.

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

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

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

Voeg het OpenAI API-key secret opnieuw toe in de productie vault, maar nu met de productiewaarde:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

Als laatste stap configureren we de decryptiesleutel op Upsun door het instellen van een sensitive variable:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Je kan alle bestanden toevoegen en committen; de decryptionkey werd automatisch aan ``.gitignore`` toegevoegd, dus deze zal nooit gecommit worden. Voor meer veiligheid kan je deze van je lokale machine verwijderen, omdat deze al gedeployd is:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Verder gaan

    * De `Symfony AI documentatie`_;

    * De `Environment Variable Processors`_;

    * `How to Keep Sensitive Information Secret`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Symfony AI documentatie`: https://symfony.com/doc/current/ai/index.html
.. _`Environment Variable Processors`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`How to Keep Sensitive Information Secret`: https://symfony.com/doc/current/configuration/secrets.html
