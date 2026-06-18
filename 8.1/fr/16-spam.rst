Empêcher le spam avec l'IA
==========================

.. index::
    single: Spam

N'importe qui peut soumettre un commentaire, même des robots ou des spammeurs. Nous pourrions ajouter un "captcha" au formulaire pour nous protéger des robots, ou nous pouvons utiliser des API tierces.

J'ai décidé d'utiliser un grand modèle de langage (LLM) pour décider si un commentaire est du spam, afin de montrer comment utiliser l'IA dans une application Symfony et comment faire de tels appels coûteux "vers l'extérieur".

Obtenir une clé d'API d'IA
--------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI prend en charge de nombreux fournisseurs de modèles : OpenAI, Anthropic, Google Gemini, Mistral, et même des modèles locaux via Ollama. Ce chapitre utilise OpenAI : créez un compte sur `platform.openai.com`_ et générez une clé d'API. Si vous préférez un autre fournisseur, le code reste le même ; seule la configuration change.

Ajouter une dépendance au Symfony AI Bundle
-------------------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Au lieu d'appeler nous-mêmes l'API HTTP du modèle, nous utiliserons le Symfony AI Bundle. Il fournit une abstraction de *plateforme* pour les fournisseurs de modèles (chaque fournisseur a son propre paquet de *bridge*) et un *agent* qui enveloppe un modèle pour effectuer les appels ; et il bénéficie de tous les outils de débogage de Symfony comme l'intégration avec le Symfony Profiler :

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI est un jeune ensemble de composants encore expérimental : ses API peuvent évoluer plus vite que le reste de Symfony.

La recette du bridge OpenAI a déjà configuré la plateforme pour nous ; elle référence une variable d'environnement ``OPENAI_API_KEY`` (et a ajouté une valeur par défaut vide pour celle-ci dans ``.env``) :

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Configurez un *agent* par défaut par-dessus :

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Utiliser des variables d'environnement
--------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Nous ne voulons certainement pas coder en dur la valeur de la clé dans la configuration ; c'est pourquoi elle est lue depuis la variable d'environnement ``OPENAI_API_KEY``.

Il appartient alors à chacun de définir une variable d'environnement "réelle" ou d'en stocker la valeur dans un fichier ``.env.local`` :

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Pour la production, une variable d'environnement "réelle" doit être définie.

Ça fonctionne bien, mais la gestion de nombreuses variables d'environnement peut devenir lourde. Dans un tel cas, Symfony a une "meilleure" alternative pour le stockage des chaînes secrètes.

Stocker des chaînes secrètes
------------------------------

.. index::
    single: Secret

Au lieu d'utiliser plusieurs variables d'environnement, Symfony peut gérer un *coffre-fort* où vous pouvez stocker plusieurs chaînes secrètes. L'une de ses caractéristiques les plus intéressantes est la possibilité de committer l'espace de stockage dans le dépôt (mais sans la clé pour l'ouvrir). Une autre fonctionnalité intéressante est qu'il peut gérer un coffre-fort par environnement.

.. index:: ! Command;secrets:set

Les chaînes secrètes sont des variables d'environnement déguisées.

Ajoutez la clé d'API d'OpenAI dans le coffre-fort :

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Comme c'est la première fois que nous exécutons cette commande, elle a généré deux clés dans le répertoire ``config/secret/dev/``. Elle a ensuite stocké la chaîne secrète ``OPENAI_API_KEY`` dans ce même répertoire.

Pour les chaînes secrètes de développement, vous pouvez décider de committer l'espace de stockage et les clés qui ont été générées dans le répertoire ``config/secret/dev/``.

Les chaînes secrètes peuvent également être écrasées en définissant une variable d'environnement du même nom.

.. index::
    single: Command;secrets:reveal

Pour relire une chaîne secrète depuis le coffre-fort, utilisez ``secrets:reveal`` :

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Concevoir une classe de vérification de spam
---------------------------------------------

.. index::
    single: AI;Prompt

Créez une nouvelle classe dans ``src/`` nommée ``SpamChecker`` pour contenir la logique qui demande au modèle si un commentaire est du spam :

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

Le *prompt système* indique au modèle son rôle et contraint ses réponses ; le *message utilisateur* contient le commentaire et le contexte de sa soumission (adresse IP, user agent).

La méthode ``getSpamScore()`` retourne 3 valeurs en fonction de la réponse du modèle :

* ``2`` : si le commentaire est un "spam flagrant" ;

* ``1`` : si le commentaire pourrait être du spam, ou si le modèle ne peut pas être joint ;

* ``0`` : si le commentaire n'est pas du spam (ham).

La sortie d'un modèle est du texte libre, même quand le prompt la contraint : analysez-la avec souplesse (passez-la en minuscules, utilisez ``str_contains()``). Et quand le modèle ne peut pas répondre du tout, repliez-vous sur la modération humaine au lieu d'échouer : l'IA doit aider l'admin, jamais bloquer le livre d'or.

.. tip::

    Essayez de soumettre un commentaire qui ressemble à du spam flagrant, comme "Buy cheap watches at http://example.com/!!!", pour voir le modèle à l'œuvre.

Identifier le spam dans les commentaires
----------------------------------------

Une façon simple de vérifier la présence de spam lorsqu'un nouveau commentaire est soumis est d'appeler le vérificateur de spam avant de stocker les données dans la base de données :

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

Vérifiez qu'il fonctionne bien.

Limiter la fréquence de soumission des commentaires
---------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Détecter le spam protège le site web contre les spammeurs sophistiqués. Une protection complémentaire et bien moins coûteuse consiste à limiter la fréquence à laquelle un même client peut soumettre des commentaires : personne ne poste légitimement des dizaines de commentaires par heure sur un livre d'or.

Ajoutez le composant Symfony Rate Limiter :

.. code-block:: terminal

    $ symfony composer req rate-limiter

Configurez un limiteur qui accepte au plus 5 commentaires par heure pour un même client :

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

Les tests automatisés soumettent légitimement beaucoup de commentaires en peu de temps, la limite est donc relevée pour l'environnement ``test``.

Appliquez le limiteur aux soumissions de commentaires avec l'attribut ``#[RateLimit]`` ; par défaut, il identifie les clients par leur adresse IP :

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

Notez l'argument ``methods`` : parcourir la page d'une conférence est une requête ``GET`` et ne doit pas être limité ; seules les soumissions de commentaires (requêtes ``POST``) le sont.

Lorsque la limite est atteinte, Symfony retourne automatiquement une réponse ``429 Too Many Requests`` avec un en-tête HTTP ``Retry-After`` indiquant au client quand il pourra réessayer.

Le même composant protège aussi le formulaire de connexion de l'admin contre les attaques par force brute ; activer le *login throttling* sur le pare-feu tient en une ligne :

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

Par défaut, Symfony bloque une IP après 5 tentatives de connexion échouées sur le même nom d'utilisateur en une minute (une connexion réussie remet le compteur à zéro). Utilisez les options ``max_attempts`` et ``interval`` pour ajuster la politique.

Gérer les chaînes secrètes en production
-------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

En production, Upsun prend en charge le paramétrage des *variables d'environnement sensibles* :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

Mais comme nous l'avons vu plus haut, l'utilisation des chaînes secrètes de Symfony pourrait être une meilleure manière de procéder. Pas en termes de sécurité, mais en termes de gestion des chaînes secrètes pour l'équipe du projet. Toutes les chaînes secrètes sont stockées dans le dépôt et la seule variable d'environnement que vous devez gérer pour la production est la clé de déchiffrement. Cela permet à tous les membres de l'équipe d'ajouter des chaînes secrètes en production même s'ils n'ont pas accès aux serveurs de production. L'installation est un peu plus compliquée cependant.

.. index::
    single: Command;secrets:generate-keys

Tout d'abord, créez une paire de clés pour l'utilisation en production :

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Rajoutez la chaîne secrète de la clé d'API d'OpenAI dans le coffre-fort en production, mais avec sa valeur de production :

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

La dernière étape consiste à envoyer la clé de déchiffrement à Upsun en définissant une variable sensible :

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Vous pouvez ajouter et commiter tous les fichiers ; la clé de déchiffrement a été ajoutée dans le ``.gitignore`` automatiquement, donc elle ne sera jamais enregistrée. Pour plus de sécurité, vous pouvez la retirer de votre machine locale puisqu'elle a été déployée :

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Aller plus loin

    * La `documentation de Symfony AI`_ ;

    * Les `processeurs de variables d'environnement`_ ;

    * `Comment garder secrètes les informations sensibles`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`documentation de Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`processeurs de variables d'environnement`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Comment garder secrètes les informations sensibles`: https://symfony.com/doc/current/configuration/secrets.html
