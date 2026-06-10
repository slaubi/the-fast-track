Adopter une méthodologie
=========================

Enseigner, c'est répéter la même chose encore et encore. Je ne le ferai pas, c'est promis. À la fin de chaque étape, vous devriez faire une petite danse et sauvegarder votre travail. C'est comme faire un ``Ctrl+S`` mais pour un site web.

Mettre en place une stratégie Git
----------------------------------

.. index::
    single: Git;add
    single: Git;commit

À la fin de chaque étape, n'oubliez pas de commiter vos modifications :

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

Vous pouvez "tout" ajouter sans risque car Symfony gère un fichier ``.gitignore`` pour vous. Et chaque paquet peut y ajouter plus de configuration. Jetez un coup d'œil au contenu actuel :

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

Les chaînes bizarres sont des marqueurs ajoutés par Symfony Flex pour qu'il sache quoi supprimer si vous décidiez de désinstaller une dépendance. Je vous l'ai dit, tout le travail fastidieux est fait par Symfony, pas vous.

Ça pourrait être une bonne idée de pusher votre dépôt vers un serveur quelque part. GitHub, GitLab ou Bitbucket sont de bons choix.

.. note::

    Si vous déployez sur Upsun, vous avez déjà une copie du dépôt Git car Upsun utilise Git de manière transparente quand vous utilisez `cloud:deploy`. Mais vous ne devriez pas vous fier au dépôt Git d'Upsun. Il est réservé pour le déploiement. Ce n'est pas une sauvegarde de votre travail.

Déploiement continu en production
----------------------------------

.. index::
    single: Symfony CLI;cloud:deploy

Une autre bonne habitude est de déployer fréquemment. Un bon rythme serait de déployer à la fin de chaque étape :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
