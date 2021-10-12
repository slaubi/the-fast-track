Voyager au cœur de Symfony
===========================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

Nous utilisons Symfony pour développer une application performante depuis un certain temps déjà, mais la plupart du code exécuté par l'application provient de Symfony. Quelques centaines de lignes de code contre des milliers de lignes de code.

J'aime comprendre comment les choses fonctionnent en coulisses. Et j'ai toujours été fasciné par les outils qui m'aident à comprendre comment fonctionnent les choses. La première fois que j'ai utilisé un débogueur pas à pas ou le moment où j'ai découvert ``ptrace`` sont des souvenirs magiques.

Vous souhaitez mieux comprendre le fonctionnement de Symfony ? Il est temps d'examiner comment Symfony fait fonctionner votre application. Au lieu de décrire comment Symfony gère une requête HTTP d'un point de vue théorique, ce qui serait assez ennuyeux, nous allons utiliser Blackfire pour obtenir quelques représentations visuelles et pour découvrir des sujets plus avancés.

Comprendre le fonctionnement interne de Symfony avec Blackfire
--------------------------------------------------------------

Vous savez déjà que toutes les requêtes HTTP sont servies par un seul point d'entrée : le fichier ``public/index.php``. Mais que se passe-t-il ensuite ? Comment sont appelés les contrôleurs ?

Profilons la page d'accueil anglaise en production avec Blackfire via l'extension de navigateur Blackfire :

.. code-block:: bash
    :class: ignore

    $ symfony remote:open

Ou directement via la ligne de commande :

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony env:urls --first`en/

Allez dans la vue "Timeline" du profil, vous devriez voir quelque chose qui ressemble à cela :

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

À partir de la timeline, passez le curseur sur les barres colorées pour avoir plus d'informations sur chaque appel, vous apprendrez beaucoup sur le fonctionnement de Symfony :

* Le point d'entrée principal est ``public/index.php`` ;

* La méthode ``Kernel::handle()`` traite la requête ;

* Il appelle le ``HttpKernel`` qui envoit des événements ;

* Le premier événement est ``RequestEvent`` ;

* La méthode ``ControllerResolver::getController()`` est appelée pour déterminer quel contrôleur doit être appelé pour l'URL entrante ;

* La méthode ``ControllerResolver::getArguments()`` est appelée pour déterminer quels arguments passer au contrôleur (le param converter est appelé) ;

* La méthode ``ConferenceController::index()`` est appelée et la majorité de notre code est exécutée par cet appel ;

* La méthode ``ConferenceRepository::findAll()`` récupère toutes les conférences de la base de données (notez la connexion à la base de données via ``PDO::__construct()``) ;

* La méthode ``Twig\Environment::render()`` génère le template ;

* Les événements ``ResponseEvent`` et ``FinishRequestEvent`` sont envoyés, mais il semble qu'aucun *listener* ne soit déclaré car ils sont exécutés très rapidement.

La timeline est un excellent moyen de comprendre le fonctionnement de certains codes, ce qui est très utile lorsque vous faites développer un projet par quelqu'un d'autre.

Maintenant, profilez la même page depuis la machine locale dans l'environnement de développement :

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

Ouvrez le profil. Vous devriez être redirigé vers l'affichage du graphique d'appel car la demande a été très rapide et la timeline sera quasiment vide :

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Comprenez-vous ce qui se passe ? Le cache HTTP est activé et, par conséquent, nous profilons la couche cache HTTP de Symfony. Comme la page est dans le cache, ``HttpCache\Store::restoreResponse()`` obtient la réponse HTTP de son cache et le contrôleur n'est jamais appelé.

Désactivez la couche cache ``public/index.php`` comme nous l'avons fait à l'étape précédente et réessayez. Vous pouvez immédiatement voir que le profil est très différent :

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

Les principales différences sont les suivantes :

* Le ``TerminateEvent``, qui n'était pas visible en production, prend un grand pourcentage du temps d'exécution. En y regardant de plus près, vous pouvez voir que c'est l'événement responsable du stockage des données nécessaires au profileur Symfony.

* Sous l'appel ``ConferenceController::index()``, remarquez la méthode ``SubRequestHandler::handle()`` qui affiche l'ESI (c'est pourquoi nous avons deux appels à ``Profiler::saveProfile()``, un pour la requête principale et un pour l'ESI).

Explorez la timeline pour en savoir plus ; passez à la vue du graphique d'appel pour avoir une représentation différente des mêmes données.

Comme nous venons de le découvrir, le code exécuté en développement et en production est assez différent. L'environnement de développement est plus lent car le profileur de Symfony essaie de rassembler beaucoup de données pour faciliter le débogage des problèmes. C'est pourquoi vous devez toujours profiler avec l'environnement de production, même au niveau local.

Quelques expériences intéressantes : profilez une page d'erreur, profilez la page ``/`` (qui est une redirection) ou une ressource API. Chaque profil vous en dira un peu plus sur le fonctionnement de Symfony, les classes/méthodes appelées, ce qui est lent à exécuter et ce qui est rapide.

Utiliser l'addon de débogage Blackfire
---------------------------------------

.. index::
    single: Blackfire;Debug Addon

Par défaut, Blackfire supprime tous les appels de méthode qui ne sont pas assez significatifs pour éviter d'avoir de grosses charges utiles et de gros graphiques. Lorsque vous utilisez Blackfire comme outil de débogage, il est préférable de conserver tous les appels. Cela est fourni par l'addon de débogage.

Depuis la ligne de commande, utilisez l'option ``--debug`` :

.. code-block:: bash
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony env:urls --first`en/

.. index::
    single: .env.local.prod

En production, vous verrez par exemple le chargement d'un fichier nommé ``.env.local.php`` :

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

D'où vient-il ? SymfonyCloud effectue certaines optimisations lors du déploiement d'une application Symfony comme l'optimisation de l'autoloader Composer (``--optimize-autoloader --apcu-autoloader --classmap-authoritative``). Il optimise également les variables d'environnement définies dans le fichier ``.env`` (pour éviter d'analyser le fichier pour chaque requête) en générant le fichier ``.env.local.php`` :

.. code-block:: bash
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire est un outil très puissant qui aide à comprendre comment le code est exécuté par PHP. L'amélioration des performances n'est qu'une raison parmi d'autres d'utiliser un profileur.
