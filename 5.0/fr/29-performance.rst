Gérer les performances
=======================

.. index::
    single: Blackfire
    single: Profiler

.. epigraph::

    L'optimisation prématurée est la racine de tous les maux.

Peut-être avez-vous déjà lu cette citation auparavant, mais j’aimerais la citer en entier :

.. epigraph::

    Nous devrions éviter les économies de bout de chandelles, disons dans environ 97 % des cas : l'optimisation prématurée est la racine de tous les maux. Pour autant, nous ne devrions pas ignorer ces occasions dans ces 3% cruciaux.

    --   Donald Knuth

Même de petites améliorations de performance peuvent faire la différence, en particulier pour les sites e-commerce. Maintenant que l'application du livre d'or est prête pour les heures de pointe, voyons comment nous pouvons analyser ses performances.

La meilleure façon de trouver des optimisations de performance est d'utiliser un *profileur*. L'option la plus populaire de nos jours est `Blackfire <https://blackfire.io>`_ (*Avertissement* : je suis aussi le fondateur du projet Blackfire).

Découvrir Blackfire
--------------------

Blackfire est composé de plusieurs parties :

* Un *client* qui déclenche des profils (l'outil Blackfire CLI ou une extension de navigateur web pour Google Chrome ou Firefox) ;

* Un *agent* qui prépare et agrège les données avant de les envoyer à blackfire.io pour affichage ;

* Une extension PHP (la *sonde*) qui analyse le code PHP.

Pour travailler avec Blackfire, vous devez d'abord vous `inscrire <https://blackfire.io/signup>`_.

Installez Blackfire sur votre machine locale en exécutant le script d'installation suivant :

.. code-block:: bash
    :class: ignore

    $ curl https://installer.blackfire.io/ | bash

Cet installateur télécharge l'outil Blackfire CLI et installe ensuite la sonde PHP (sans l'activer) sur toutes les versions PHP disponibles.

Activez la sonde PHP pour notre projet :

.. code-block:: diff
    :caption: patch_file

    --- a/php.ini
    +++ b/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

Redémarrez le serveur web pour que PHP puisse charger Blackfire :

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

L'outil Blackfire CLI doit être configuré avec vos identifiants **client** personnels (pour stocker vos profils de projet dans votre compte personnel). Vous les trouverez en haut de la `page <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials``. Exécutez la commande suivante en remplaçant les espaces réservés :

.. code-block:: bash
    :class: ignore

    $ blackfire config --client-id=xxx --client-token=xxx

.. note::

    Pour des instructions d'installation complètes, suivez le `guide d'installation officiel détaillé <https://blackfire.io/docs/up-and-running/installation>`_. Elles sont utiles lors de l'installation de Blackfire sur un serveur.

Configurer l'agent Blackfire sur Docker
---------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

La dernière étape consiste à ajouter le service d'agent Blackfire dans Docker Compose :

.. code-block:: diff
    :caption: patch_file

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -20,3 +20,8 @@ services:
         mailer:
             image: schickling/mailcatcher
             ports: [1025, 1080]
    +
    +    blackfire:
    +        image: blackfire/blackfire
    +        env_file: .env.local
    +        ports: [8707]

Pour communiquer avec le serveur, vous devez récupérer vos identifiants de **serveur** personnels (ces identifiants spécifient l'endroit où vous voulez stocker les profils - vous pouvez en créer un par projet) ; ils se trouvent au bas de la `page <https://blackfire.io/my/settings/credentials>`_ ``Settings/Credentials``. Stockez-les dans un fichier local ``.env.local`` :

.. code-block:: text
    :class: ignore

    BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Vous pouvez maintenant lancer le nouveau conteneur :

.. code-block:: bash
    :class: ignore

    $ docker-compose stop
    $ docker-compose up -d

Réparer une installation Blackfire en panne
--------------------------------------------

Si vous obtenez une erreur pendant le profilage, augmentez le niveau de log Blackfire pour obtenir plus d'informations :

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/php.ini
    +++ b/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

Redémarrez le serveur web :

.. code-block:: bash
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Et suivez les logs :

.. code-block:: bash
    :class: ignore

    $ symfony server:log

Profilez à nouveau et vérifiez les logs.


Configurer Blackfire en production
----------------------------------

.. index::
    single: SymfonyCloud;Blackfire

Blackfire est inclus par défaut dans tous les projets SymfonyCloud.

Configurez les identifiants du *serveur* comme variables d'environnement :

.. code-block:: bash
    :class: ignore

    $ symfony var:set BLACKFIRE_SERVER_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony var:set BLACKFIRE_SERVER_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Et activez la sonde PHP comme n'importe quelle autre extension PHP :

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - blackfire
             - xsl
             - amqp
             - redis

Configurer Varnish pour Blackfire
---------------------------------

.. index::
    single: SymfonyCloud;Varnish

Avant de pouvoir déployer pour commencer le profilage, vous devez trouver un moyen de contourner le cache HTTP de Varnish. Sinon, Blackfire n'atteindra jamais l'application PHP. Vous n'autoriserez que les demandes de profil provenant de votre machine locale.

Récupérez votre adresse IP actuelle :

.. code-block:: bash
    :class: ignore

    $ curl https://ifconfig.me/

Et utilisez-la pour configurer Varnish :

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "a.b.c.d";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

Vous pouvez maintenant déployer.

Profiler les pages web
----------------------

.. index::
    single: Profiling;Web Pages

Vous pouvez profiler les pages web traditionnelles depuis Firefox ou Google Chrome grâce à leurs `extensions dédiées <https://blackfire.io/docs/integrations/browsers/index>`_.

Sur votre machine locale, n'oubliez pas de désactiver le cache HTTP dans ``public/index.php`` lors du profilage : sinon, vous profilerez la couche de cache HTTP Symfony au lieu de votre propre code :

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- a/public/index.php
    +++ b/public/index.php
    @@ -24,7 +24,7 @@ if ($trustedHosts = $_SERVER['TRUSTED_HOSTS'] ?? $_ENV['TRUSTED_HOSTS'] ?? false
     $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);

     if ('dev' === $kernel->getEnvironment()) {
    -    $kernel = new HttpCache($kernel);
    +//    $kernel = new HttpCache($kernel);
     }

     $request = Request::createFromGlobals();

Pour avoir une meilleure idée de la performance de votre application en production, vous devez également profiler l'environnement "production". Par défaut, votre environnement local utilise l'environnement de "développement", ce qui ajoute un surcoût important (principalement pour collecter des données pour la web debug toolbar et le profileur Symfony).

.. index::
    single: Symfony CLI;server:prod

Le passage de votre machine locale à l'environnement de production peut se faire en changeant la variable d'environnement ``APP_ENV`` dans le fichier ``.env.local`` :

.. code-block:: text
    :class: ignore

    APP_ENV=prod

Ou vous pouvez utiliser la commande ``server:prod`` :

.. code-block:: bash
    :class: ignore

    $ symfony server:prod

N'oubliez pas de le remettre sur dev à la fin de votre session de profilage :

.. code-block:: bash
    :class: ignore

    $ symfony server:prod --off

Profiler les ressources de l'API
--------------------------------

.. index::
    single: Profiling;API

Le profilage de l'API ou de la SPA est plus efficace en ligne de commande, en utilisant l'outil Blackfire CLI que vous avez installé précédemment :

.. code-block:: bash
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

La commande ``blackfire curl`` accepte exactement les mêmes arguments et options que `cURL <https://curl.haxx.se/docs/manpage.html>`_.

Comparer les performances
-------------------------

Dans l'étape traitant du "Cache", nous avons ajouté une couche cache pour améliorer les performances de notre code, mais nous n'avons pas vérifié ni mesuré l'impact du changement sur les performances. Comme nous sommes tous très mauvais pour deviner ce qui sera rapide et ce qui est lent, vous pourriez vous retrouver dans une situation où l'optimisation rend votre application plus lente.

Vous devriez toujours mesurer l'impact de toute optimisation que vous faites avec un profileur. Blackfire facilite l'analyse grâce à sa `fonction de comparaison <https://blackfire.io/docs/cookbooks/understanding-comparisons>`_.

Écrire les tests fonctionnels de boîte noire
----------------------------------------------

.. index::
    single: Blackfire;Player

Nous avons vu comment écrire des tests fonctionnels avec Symfony. Blackfire peut être utilisé pour écrire des scénarios de navigation qui peuvent être exécutés à la demande via le `lecteur Blackfire <https://blackfire.io/player>`_. Rédigeons un scénario qui soumet un nouveau commentaire et le valide via le lien email en développement, et via l'interface d'admin en production.

Créez un fichier ``.blackfire.yaml`` avec le contenu suivant :

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment_form[author] 'Fabien'
                param comment_form[email] 'me@example.com'
                param comment_form[text] 'Such a good conference!'
                param comment_form[photo] file(fake('image', '/tmp', 400, 300, 'cats'), 'awesome-cat.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin/?entity=Comment&action=list')
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Téléchargez le lecteur Blackfire pour pouvoir exécuter le scénario localement :

.. code-block:: bash

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

Exécutez ce scénario en développement :

.. code-block:: bash

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev"

Ou en production :

.. code-block:: bash
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony env:urls --first` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod"

Les scénarios Blackfire peuvent également déclencher des profils pour chaque requête et exécuter des tests de performance en ajoutant l'option ``--blackfire``.

Automatiser les contrôles de performance
-----------------------------------------

La gestion de la performance ne consiste pas seulement à améliorer la performance du code existant, mais aussi à vérifier qu'aucune régression de performance n'est introduite.

Le scénario décrit dans la section précédente peut être exécuté automatiquement dans un workflow d'intégration continue ou régulièrement en production.

SymfonyCloud permet également d'`exécuter les scénarios <https://blackfire.io/docs/integrations/paas/symfonycloud#builds-level-enterprise>`_ à chaque fois que vous créez une nouvelle branche ou déployez en production pour vérifier automatiquement les performances du nouveau code.

.. sidebar:: Aller plus loin

    * `Le livre Blackfire : PHP Code Performance Explained <https://blackfire.io/book>`_ ;

    * `Tutoriel SymfonyCasts sur Blackfire <https://symfonycasts.com/screencast/blackfire>`_.
