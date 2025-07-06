De zéro à la production
=========================

J'aime aller vite et je veux que notre petit projet soit réalisé le plus rapidement possible. Du genre en production, dès maintenant. Comme nous n'avons encore rien développé, nous allons commencer par déployer une simple page "En construction". Vous allez adorer !

Passez un peu de temps à chercher sur internet un GIF animé "En construction" bien démodé. Voici `celui`_ que je vais utiliser :

.. image:: images/under-construction.gif
    :align: center

Je vous l'ai dit, nous allons bien nous amuser.

Initialiser le projet
---------------------

Créez un nouveau projet Symfony avec la commande ``symfony`` que nous avons installée ensemble auparavant :

.. code-block:: terminal

    $ symfony new guestbook --version=6.4 --php=8.3 --webapp --docker --cloud
    $ cd guestbook

Cette commande est une mince surcouche de ``Composer`` qui facilite la création de projets Symfony. Elle utilise un `squelette de projet`_ qui inclut uniquement les composants Symfony requis par presque tous les projets : un outil console et l'abstraction HTTP nécessaire pour créer des applications web.

Comme nous créans une application web complète, nous avons ajouté quelques options qui nous faciliterons la vie :

* ``--webapp``: Par défaut, une application avec le moins de dépendances possibles est créée. Pour la plupart des projets web, il est recommandé d'utiliser le paquet ``webapp``. Il contient la majeure partie des paquets nécessaires pour une application web "moderne". Le paquet "webapp" ajoute un nombre important de paquets Symfony, comme Symfony Messenger et PostgreSQL via Doctrine.

* ``--docker`` : Sur votre machine locale, nous utiliserons Docker pour gérer les services comme PostgreSQL. Cette option active Docker de manière à ce que Symfony ajoute de manière automatique les services Docker requis par les paquets installés (un service PostgreSQL quand l'ORM est ajouté ou un intercepteur de mails quand Symfony Mailer est ajouté par exemple).

* ``--cloud`` : Si vous voulez déployer sur Platform.sh, cette option génère automatiquement une configuration pour Platform.sh. Platform.sh est la manière la plus simple de déployer les environnements de test, préprod et production

Si vous regardez le dépôt GitHub pour le squelette, vous remarquerez qu'il est presque vide : il ne contient qu'un fichier ``composer.json``, mais le répertoire ``guestbook`` est lui plein de fichiers. Comment est-ce possible ? La réponse se trouve dans le paquet ``symfony/flex``. Symfony Flex est un plugin Composer qui se greffe au processus d'installation. Lorsqu'il détecte un paquet pour lequel une *recette* existe, il l'exécute.

Le point d'entrée principal d'une recette Symfony est un fichier *manifest* qui décrit les opérations à effectuer pour intégrer automatiquement le paquet dans l'application. Vous n'avez jamais besoin de lire un fichier *README* pour installer un paquet avec Symfony. L'automatisation est au cœur de Symfony.

Comme Git est installé sur notre machine, ``symfony new`` nous a également créé un dépôt Git, dans lequel a été ajouté le tout premier commit.

Jetons un coup d'oeil à la structure des répertoires :

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

Le répertoire ``bin/`` contient le principal point d'entrée de la ligne de commande : ``console``. Vous l'utiliserez tout le temps.

Le répertoire ``config/`` est constitué d'un ensemble de fichiers de configuration sensibles, initialisés avec des valeurs par défaut. Un fichier par paquet. Vous les modifierez rarement : faire confiance aux valeurs par défaut est presque toujours une bonne idée.

Le répertoire ``public/`` est le répertoire racine du site web, et le script ``index.php`` est le point d'entrée principal de toutes les ressources HTTP dynamiques.

Le répertoire ``src/`` héberge tout le code que vous allez écrire ; c'est ici que vous passerez la plupart de votre temps. Par défaut, toutes les classes de ce répertoire utilisent le *namespace* PHP ``App``. C'est votre répertoire de travail, votre code, votre logique de domaine. Symfony n'a pas grand-chose à y faire.

Le répertoire ``var/`` contient les caches, les logs et les fichiers générés par l'application lors de son exécution. Vous pouvez le laisser tranquille. C'est le seul répertoire qui doit être en écriture en production.

Le répertoire ``vendor/`` contient tous les paquets installés par Composer, y compris Symfony lui-même. C'est notre arme secrète pour un maximum de productivité. Ne réinventons pas la roue. Vous profiterez des bibliothèques existantes pour vous faciliter le travail. Le répertoire est géré par Composer. N'y touchez jamais.

C'est tout ce que vous avez besoin de savoir pour l'instant.

Créer des ressources publiques
-------------------------------

Tout ce qui se trouve dans le répertoire ``public/`` est accessible par un navigateur. Par exemple, si vous déplacez votre fichier GIF animé (nommez-le ``under-construction.gif``) dans un nouveau répertoire ``public/images/``, il sera alors disponible à une URL comme ``https://localhost/images/under-construction.gif``.

Téléchargez mon image GIF ici :

.. code-block:: terminal

    $ mkdir public/images/
    $ php -r "copy('https://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Lancer un serveur web local
---------------------------

.. index::
    single: Symfony CLI;server:start

La commande ``symfony`` inclut un serveur web optimisé pour le développement. Comme vous vous en doutez, il marche très bien avec Symfony. Cependant, ne l'utilisez jamais en production.

À partir du répertoire du projet, démarrez le serveur web en arrière-plan (option ``-d``) :

.. code-block:: terminal

    $ symfony server:start -d

Le serveur a démarré sur le premier port disponible (à partir de 8000). Pour gagner du temps, vous pouvez ouvrir le site web dans un navigateur à partir de la ligne de commande :

.. code-block:: terminal
    :class: ignore

    $ symfony open:local

Votre navigateur favori devrait recevoir le focus et ouvrir un nouvel onglet affichant une page similaire à celle-ci :

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    Pour résoudre les problèmes, exécutez ``symfony server:log`` ; cette commande affiche les derniers logs de votre serveur web, de PHP et de votre application.

Naviguez vers ``/images/under-construction.gif``. Est-ce que cela ressemble à ceci ?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

Tout est bon ? Commitons notre travail :

.. code-block:: terminal
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Se préparer pour la production
-------------------------------

.. index::
    single: Platform.sh;Initialization

Qu'en est-il du déploiement de notre travail en production ? Je sais, nous n'avons même pas encore de page HTML pour accueillir convenablement nos internautes, mais voir la petite image "en construction" sur un serveur de production serait une grande satisfaction. Et vous connaissez la devise : *déployer tôt, déployez souvent*.

Vous pouvez héberger cette application chez n'importe quel fournisseur supportant PHP, soit presque tous les hébergeurs. Vérifiez tout de même quelques points : nous voulons la dernière version de PHP et la possibilité d'héberger des services comme une base de données, une file d'attente, etc.

J'ai fait mon choix, ce sera `Platform.sh`_. Il fournit tout ce dont nous avons besoin et aide à financer le développement de Symfony.

.. index::
    single: Symfony CLI;project:init

Comme nous avons utilisé l'option ``--cloud`` quand nous avons créé le projet, Platform.sh a déjà été initialisé avec les quelques fichiers requis par Platform.sh, comme ``.platform/services.yaml``, ``.platform/routes.yaml`` et ``.platform.app.yaml``.

Mise en production
------------------

.. index::
    single: Symfony CLI;cloud:project:create
    single: Symfony CLI;cloud:deploy

On déploie ?

Créez un nouveau projet Platform.sh distant :

.. code-block:: terminal

    $ symfony cloud:project:create --title="Guestbook" --plan=development

Cette commande fait beaucoup de choses :

* La première fois que vous lancez cette commande, identifiez-vous avec votre compte Platform.sh, si ce n'était pas déjà fait.

* Elle crée un nouveau projet sur Platform.sh (vous bénéficiez de 30 jours *gratuits* sur le premier projet que vous créez).

Puis, déployez :

.. code-block:: terminal

    $ symfony cloud:deploy

Le code est déployé en pushant le dépôt Git. À la fin de la commande, le projet sera accessible par un nom de domaine précis.

.. index::
    single: Symfony CLI;cloud:url

Vérifiez que tout fonctionne bien :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Vous devriez obtenir une 404, mais si vous naviguez vers ``/images/under-construction.gif`` vous devriez pouvoir admirer votre travail.

Notez que vous n'obtenez pas la belle page par défaut de Symfony sur Platform.sh. Pourquoi ? Vous apprendrez bientôt que Symfony supporte plusieurs environnements, et que Platform.sh a automatiquement déployé le code en environnement de production.

.. index::
    single: Symfony CLI;cloud:project:delete

.. tip::

    Si vous voulez supprimer le projet sur Platform.sh, utilisez la commande ``cloud:project:delete``.

.. sidebar:: Aller plus loin

    * Les dépôts pour les `recettes officielles de Symfony`_ et pour les `recettes créées par la communauté`_, où vous pouvez soumettre vos propres recettes ;

    * Le `serveur web local de Symfony`_ ;

    * La `documentation de Platform.sh`_.

.. _`celui`: https://clipartmag.com/images/website-under-construction-image-6.gif
.. _`squelette de projet`: https://github.com/symfony/skeleton
.. _`Platform.sh`:     https://platform.sh/marketplace/symfony/?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
.. _`recettes officielles de Symfony`: https://github.com/symfony/recipes
.. _`recettes créées par la communauté`: https://github.com/symfony/recipes-contrib
.. _`serveur web local de Symfony`: https://symfony.com/doc/current/setup/symfony_server.html
.. _`documentation de Platform.sh`: https://docs.platform.sh/guides/symfony.html?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
