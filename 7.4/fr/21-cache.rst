Mettre en cache pour la performance
===================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Les problèmes de performance peuvent survenir avec la popularité. Quelques exemples typiques : des index de base de données manquants ou des tonnes de requêtes SQL par page. Vous n'aurez aucun problème avec une base de données vide, mais avec plus de trafic et des données croissantes, cela peut arriver à un moment donné.

Ajouter des en-têtes de cache HTTP
-----------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

L'utilisation de stratégies de mise en cache HTTP est un excellent moyen de maximiser les performances de notre site avec un minimum d'effort. Ajoutez un cache reverse proxy en production pour permettre la mise en cache et utilisez un `CDN`_  pour aller encore plus loin.

Mettons en cache la page d'accueil pendant une heure :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -29,7 +29,7 @@ final class ConferenceController extends AbstractController
         {
             return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

La méthode ``setSharedMaxAge()`` configure l'expiration du cache pour les reverse proxies. Utiliser ``setMaxAge()`` permet de contrôler le cache du navigateur. Le temps est exprimé en secondes (1 heure = 60 minutes = 3600 secondes).

La mise en cache de la page de la conférence est plus difficile car elle est plus dynamique. N'importe qui peut ajouter un commentaire à tout moment, et personne ne veut attendre une heure pour le voir en ligne. Dans de tels cas, utilisez la stratégie de *validation HTTP*.

Activer le noyau de cache HTTP de Symfony
-----------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Pour tester la stratégie de cache HTTP, activez le reverse proxy HTTP de Symfony, mais seulement dans l'environnement de "développement" (pour l'environnement de "production", nous utiliserons une solution "plus robuste") :

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

En plus d'être un véritable reverse proxy HTTP, le reverse proxy HTTP de Symfony (via la classe ``HttpCache``) ajoute quelques informations de débogage sous forme d'en-têtes HTTP. Cela aide grandement à valider les en-têtes de cache que nous avons définis.

Vérifiez sur la page d'accueil :

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

Pour la toute première requête, le serveur de cache vous indique que c'était un ``miss`` et qu'il a exécuté une action de ``store`` pour mettre la réponse en cache. Vérifiez l'en-tête ``cache-control`` pour voir la stratégie de cache configurée.

Pour les prochaines demandes, la réponse est mise en cache (l'``age`` a également été mis à jour) :

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

Éviter des requêtes SQL avec les ESIs
---------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

Le *listener* ``TwigEventSubscriber`` injecte une variable globale dans Twig avec tous les objets de conférence, et ce sur chaque page du site web. C'est probablement une excellente chose à optimiser.

Vous n'ajouterez pas de nouvelles conférences tous les jours, donc le code interroge la base de données pour récupérer exactement les mêmes données encore et encore.

Nous pourrions vouloir mettre en cache les noms et les *slugs* des conférences avec le cache Symfony, mais dès que possible, j'aime me reposer sur le système de mise en cache HTTP.

Lorsque vous voulez mettre en cache un fragment d'une page, déplacez-le en dehors de la requête HTTP en cours en créant une *sous-requête*. *ESI* correspond parfaitement à ce cas d'utilisation. Un ESI est un moyen d'intégrer le résultat d'une requête HTTP dans une autre.

Créez un contrôleur qui ne renvoie que le fragment HTML qui affiche les conférences :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -32,6 +32,14 @@ final class ConferenceController extends AbstractController
             ])->setSharedMaxAge(3600);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,

Créez le template correspondant :

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Interrogez la route ``/conference_header`` pour vérifier que tout fonctionne bien.

.. index::
    single: Twig;render
    single: Twig;path

Il est temps de dévoiler l'astuce ! Mettez à jour le template Twig pour appeler le contrôleur que nous venons de créer :

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Et voilà. Rafraîchissez la page et le site web affiche toujours la même chose.

.. tip::

    Utilisez le panneau du profileur Symfony "Request / Response" pour en savoir plus sur la requête principale et ses sous-requêtes.

Maintenant, chaque fois que vous affichez une page dans le navigateur, deux requêtes HTTP sont exécutées : une pour l'en-tête et une pour la page principale. Vous avez dégradé les performances. Félicitations !

L'appel HTTP pour l'en-tête est actuellement effectué en interne par Symfony, donc aucun aller-retour HTTP n'est impliqué. Cela signifie également qu'il n'y a aucun moyen de bénéficier des en-têtes de cache HTTP.

Convertissez l'appel en un "vrai" appel HTTP à l'aide d'un ESI.

Tout d'abord, activez le support ESI :

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Ensuite, utilisez ``render_esi`` au lieu de ``render`` :

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Si Symfony détecte un reverse proxy qui sait comment traiter les ESIs, il active automatiquement le support (sinon, par défaut, il génère le rendu de la sous-demande de manière synchrone).

Comme le reverse proxy de Symfony supporte les ESIs, vérifions ses logs (supprimons d'abord le cache - voir "Purger le cache" ci-dessous) :

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

Rafraîchissez quelques fois : la réponse à la route``/`` est mise en cache et celle à ``/conference_header`` ne l'est pas. Nous avons réalisé quelque chose de génial : toute la page est dans le cache mais elle conserve toujours une partie dynamique.

Mais ce n'est pas ce que nous voulons. Mettez l'en-tête de la page en cache pendant une heure, indépendamment de tout le reste :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -37,7 +37,7 @@ final class ConferenceController extends AbstractController
         {
             return $this->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]);
    +        ])->setSharedMaxAge(3600);
         }

         #[Route('/conference/{slug}', name: 'conference')]

Le cache est maintenant activé pour les deux requêtes :

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

L'en-tête ``x-symfony-cache`` contient deux éléments : la requête principale ``/`` et une sous-requête (l'ESI ``conference_header``). Les deux sont dans le cache (``fresh``).

La stratégie de cache peut être différente entre la page principale et ses ESIs. Si nous avons une page "about", nous pourrions vouloir la stocker pendant une semaine dans le cache, tout en ayant l'en-tête mis à jour toutes les heures.

Supprimez le listener car nous n'en avons plus besoin :

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Purger le cache HTTP pour les tests
-----------------------------------

Tester le site web dans un navigateur ou via des tests automatisés devient un peu plus difficile avec une couche de cache.

Vous pouvez supprimer manuellement tout le cache HTTP en supprimant le répertoire ``var/cache/dev/http_cache/`` :

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

Cette stratégie ne fonctionne pas bien si vous voulez seulement invalider certaines URLs ou si vous voulez intégrer l'invalidation du cache dans vos tests fonctionnels. Ajoutons un petit point d'entrée HTTP, réservé à l'admin, pour invalider certaines URLs :

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

Le nouveau contrôleur a été limité à la méthode HTTP ``PURGE``. Cette méthode n'est pas dans le standard HTTP, mais elle est largement utilisée pour invalider les caches.

Par défaut, les paramètres de routage ne peuvent pas contenir ``/`` car ils séparent les segments d'une URL. Vous pouvez remplacer cette restriction pour le dernier paramètre de routage, comme ``uri`` par exemple, en définissant votre propre masque (``.*``).

La manière par laquelle nous obtenons l'instance ``HttpCache`` peut aussi sembler un peu étrange ; nous utilisons une classe anonyme, car l'accès à la classe "réelle" n'est pas possible. L'instance ``HttpCache`` enveloppe le noyau réel, qui n'est volontairement pas conscient de la couche de cache.

Invalidez la page d'accueil et l'en-tête avec les conférences via les appels cURL suivants :

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

La sous-commande ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` retourne l'URL courante du serveur web local.

.. note::

    Le contrôleur n'a pas de nom de route car il ne sera jamais référencé dans le code.

Regrouper les routes similaires avec un préfixe
------------------------------------------------

.. index::
    single: Attributes;Route

Les deux routes du contrôleur admin ont le même préfixe ``/admin``. Au lieu de le répéter sur toutes les routes, refactorisez-les pour configurer le préfixe sur la classe elle-même :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

Mettre en cache les opérations coûteuses en CPU/mémoire
----------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

Nous n'avons pas d'algorithmes gourmands en CPU ou en mémoire sur le site web. Pour parler des *caches locaux*, créons une commande qui affiche l'étape en cours sur laquelle nous travaillons (pour être plus précis, le nom du tag Git attaché au commit actuel).

Le composant Symfony Process vous permet d'exécuter une commande et de récupérer le résultat (sortie standard et erreur).

Créez la commande :

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand extends Command
    {
        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Vous auriez pu utiliser ``make:command`` pour créer la commande :

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

Et si on veut mettre le résultat en cache pendant quelques minutes ? Utilisez le cache Symfony.

Et insérez le code dans la logique de cache :

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -7,15 +7,27 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand extends Command
     {
    +    public function __construct(
    +         private CacheInterface $cache,
    +    ) {
    +        parent::__construct();
    +    }
    +
         protected function execute(InputInterface $input, OutputInterface $output): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $this->cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

Le processus n'est maintenant appelé que si l'élément ``app.current_step`` n'est pas dans le cache.

Analyser et comparer les performances
-------------------------------------

N'ajoutez jamais de cache à l'aveuglette. Gardez à l'esprit que l'ajout d'un cache ajoute une couche de complexité. Et comme nous sommes tous très mauvais pour deviner ce qui sera rapide et ce qui est lent, vous pourriez vous retrouver dans une situation où le cache rend votre application plus lente.

Mesurez toujours l'impact de l'ajout d'un cache avec un outil de profilage comme `Blackfire`_.

Reportez-vous à l'étape "Performances" pour en savoir plus sur la façon dont vous pouvez utiliser Blackfire pour tester votre code avant de le déployer.

Configurer un cache de reverse proxy en production
--------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

Plutôt que d'utiliser le reverse proxy Symfony en production, nous allons utiliser Varnish, un reverse proxy "plus robuste".

Ajoutez Varnish aux services Upsun :

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -6,6 +6,15 @@ services:
         database:
             type: postgresql:16

    +    varnish:
    +        type: varnish:7.6
    +        relationships:
    +            application: 'app:http'
    +        configuration:
    +            vcl: !include
    +                type: string
    +                path: config.vcl
    +
     applications:

.. index::
    single: Upsun;Routes

Utilisez Varnish comme point d'entrée principal dans les routes :

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

Enfin, créez un fichier ``config.vcl`` pour configurer Varnish :

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Activer le support ESI sur Varnish
----------------------------------

La prise en charge des ESIs sur Varnish devrait être activée explicitement pour chaque requête. Pour le rendre global, Symfony utilise les en-têtes standard ``Surrogate-Capability`` et ``Surrogate-Control`` pour activer le support ESI :

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Purger le cache de Varnish
--------------------------

L'invalidation du cache en production ne devrait probablement jamais être nécessaire, sauf en cas d'urgence, et peut-être si vous n'êtes pas dans la branche ``master``. Si vous avez besoin de souvent purger le cache, cela signifie probablement que la stratégie de mise en cache doit être modifiée (en réduisant le TTL, ou en utilisant une stratégie de validation au lieu d'une stratégie d'expiration).

Quoi qu'il en soit, voyons comment configurer Varnish pour l'invalidation du cache :

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

Dans la vraie vie, vous restreindriez probablement plutôt par IPs comme décrit dans la `documentation de Varnish`_.

Purgez quelques URLs maintenant :

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

Les URLs semblent un peu étranges parce que celles renvoyées par ``env:url`` se terminent déjà par ``/``.

.. sidebar:: Aller plus loin

    * `Cloudflare`_, la plate-forme cloud globale ;

    * `Documentation du cache HTTP de Varnish`_ ;

    * `Spécifications ESI`_ et `ressources ESI`_ ;

    * `Modèle de validation de cache HTTP`_ ;

    * `Cache HTTP dans Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`documentation de Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Documentation du cache HTTP de Varnish`: https://varnish-cache.org/docs/index.html
.. _`Spécifications ESI`: https://www.w3.org/TR/esi-lang
.. _`ressources ESI`: https://www.akamai.com/us/en/support/esi.jsp
.. _`Modèle de validation de cache HTTP`: https://symfony.com/doc/current/http_cache/validation.html
.. _`Cache HTTP dans Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
