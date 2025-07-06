Empêcher le spam avec une API
==============================

.. index::
    single: Spam

N'importe qui peut soumettre un commentaire, même des robots ou des spammeurs. Nous pourrions ajouter un "captcha" au formulaire pour nous protéger des robots, ou nous pouvons utiliser des API tierces.

J'ai décidé d'utiliser le service gratuit `Akismet`_ pour montrer comment appeler une API et comment faire un appel "vers l'extérieur".

S'inscrire sur Akismet
----------------------

.. index::
    single: Akismet

Créez un compte gratuit sur `akismet.com`_ et récupérez la clé de l'API Akismet.

Ajouter une dépendance au composant Symfony HTTPClient
-------------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Au lieu d'utiliser une bibliothèque qui abstrait l'API d'Akismet, nous ferons directement tous les appels API. Faire nous-mêmes les appels HTTP est plus efficace (et nous permet de bénéficier de tous les outils de débogage de Symfony comme l'intégration avec le Symfony Profiler).

Concevoir une classe de vérification de spam
---------------------------------------------

Créez une nouvelle classe dans ``src/`` nommée ``SpamChecker`` pour contenir la logique d'appel à l'API d'Akismet et l'interprétation de ses réponses :

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

La méthode ``request()`` du client HTTP soumet une requête POST à l'URL d'Akismet (``$this->endpoint``) et passe un tableau de paramètres.

La méthode ``getSpamScore()`` retourne 3 valeurs en fonction de la réponse de l'appel à l'API :

* ``2`` : si le commentaire est un "spam flagrant" ;

* ``1`` : si le commentaire pourrait être du spam ;

* ``0`` : si le commentaire n'est pas du spam (ham).

.. tip::

    Utilisez l'adresse email spéciale ``akismet-guaranteed-spam@example.com`` pour forcer le résultat de l'appel à être du spam.

Utiliser des variables d'environnement
--------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

La classe ``SpamChecker`` utilise un argument ``$akismetKey``. Comme pour le répertoire d'upload, nous pouvons l'injecter grâce à l'attribut ``Autowire`` du conteneur :

.. code-block:: diff
    :caption: patch_file

    --- i/src/SpamChecker.php
    +++ w/src/SpamChecker.php
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

Nous ne voulons certainement pas coder en dur la valeur de la clé d'Akismet dans le code, nous utilisons donc plutôt une variable d'environnement (``AKISMET_KEY``).

Il appartient alors à chacun de définir une variable d'environnement "réelle" ou d'en stocker la valeur dans un fichier ``.env.local`` :

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Pour la production, une variable d'environnement "réelle" doit être définie.

Ça fonctionne bien, mais la gestion de nombreuses variables d'environnement peut devenir lourde. Dans un tel cas, Symfony a une "meilleure" alternative pour le stockage des chaînes secrètes.

Stocker des chaînes secrètes
------------------------------

.. index::
    single: Secret

Au lieu d'utiliser plusieurs variables d'environnement, Symfony peut gérer un *coffre-fort* où vous pouvez stocker plusieurs chaînes secrètes. L'une de ses caractéristiques les plus intéressantes est la possibilité de committer l'espace de stockage dans le dépôt (mais sans la clé pour l'ouvrir). Une autre fonctionnalité intéressante est qu'il peut gérer un coffre-fort par environnement.

.. index:: ! Command;secrets:set

Les chaînes secrètes sont des variables d'environnement déguisées.

Ajoutez la clé d'Akismet dans le coffre-fort :

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Comme c'est la première fois que nous exécutons cette commande, elle a généré deux clés dans le répertoire ``config/secret/dev/``. Elle a ensuite stocké la chaîne secrète ``AKISMET_KEY`` dans ce même répertoire.

Pour les chaînes secrètes de développement, vous pouvez décider de committer l'espace de stockage et les clés qui ont été générées dans le répertoire ``config/secret/dev/``.

Les chaînes secrètes peuvent également être écrasées en définissant une variable d'environnement du même nom.

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
    @@ -34,6 +35,7 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
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

Gérer les chaînes secrètes en production
-------------------------------------------

.. index::
    single: Platform.sh;Secret
    single: Platform.sh;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

En production, Platform.sh prend en charge le paramétrage des *variables d'environnement sensibles* :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

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

Rajoutez la chaîne secrète d'Akismet dans le coffre-fort en production, mais avec sa valeur de production :

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

La dernière étape consiste à envoyer la clé de déchiffrement à Platform.sh en définissant une variable sensible :

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Vous pouvez ajouter et commiter tous les fichiers ; la clé de déchiffrement a été ajoutée dans le ``.gitignore`` automatiquement, donc elle ne sera jamais enregistrée. Pour plus de sécurité, vous pouvez la retirer de votre machine locale puisqu'elle a été déployée :

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Aller plus loin

    * La `documentation du composant HttpClient`_ ;

    * Les `processeurs de variables d'environnement`_ ;

    * La `cheat sheet du HttpClient de Symfony`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`documentation du composant HttpClient`: https://symfony.com/doc/current/components/http_client.html
.. _`processeurs de variables d'environnement`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`cheat sheet du HttpClient de Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
