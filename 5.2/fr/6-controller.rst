Créer un contrôleur
=====================

.. index::
    single: Controller
    single: Routing;Route

Notre projet de livre d'or est déjà en ligne sur les serveurs de production, mais nous avons un peu triché. Le projet n'a pas encore de page web. La page d'accueil est une ennuyeuse page d'erreur 404. Corrigeons cela.

Lorsqu'une requête HTTP arrive au serveur, comme pour notre page d'accueil (``http://localhost:8000/``), Symfony essaie de trouver une *route* qui corresponde au *chemin de la requête* (``/`` ici). Une *route* est le lien entre le chemin de la requête et un *callable PHP*, une fonction devant créer la *réponse* HTTP associée à cette requête.

Ces *callables* sont nommés "contrôleurs". Dans Symfony, la plupart des contrôleurs sont implémentés sous la forme de classes PHP. Vous pouvez créer ces classes manuellement, mais comme nous aimons aller vite, voyons comment Symfony peut nous aider.

Se faciliter la vie avec le *Maker Bundle*
------------------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Pour générer des contrôleurs facilement, nous pouvons utiliser le paquet ``symfony/maker-bundle`` :

.. code-block:: bash

    $ symfony composer req maker --dev

Comme le *Maker Bundle* n'est utile que pendant le développement, n'oubliez pas d'ajouter l'option ``--dev`` pour éviter qu'il ne soit activé en production.

Le *Maker Bundle* vous permet de générer un grand nombre de classes différentes. Nous l'utiliserons constamment dans ce livre. Chaque "générateur" correspond à une commande et chacune d'entre elles appartient au même *namespace* ``make``.

.. index::
    single: Command;list

La commande ``list``, intégrée nativement à la console ``symfony``, permet d'afficher toutes les commandes disponibles sous un *namespace* donné ; utilisez-la pour découvrir tous les générateurs fournis par le *Maker Bundle* :

.. code-block:: bash
    :class: ignore

    $ symfony console list make

Choisir un format de configuration
----------------------------------

Avant de créer le premier contrôleur du projet, nous devons décider des formats de configuration que nous voulons utiliser. Symfony supporte nativement YAML, XML, PHP et les annotations.

Pour la *configuration des paquets*, *YAML* est le meilleur choix. C'est le format utilisé dans le répertoire ``config/``. Souvent, lorsque vous installez un nouveau paquet, la recette de ce paquet crée un nouveau fichier se terminant par ``.yaml`` dans ce répertoire.

Pour la *configuration liée au code PHP*, les *annotations* sont plus appropriées, car elles cohabitent avec le code. Prenons un exemple : lorsqu'une requête arrive, la configuration doit indiquer à Symfony que le chemin de la requête doit être géré par un contrôleur spécifique (une classe PHP). Si notre configuration est en YAML, XML ou PHP, deux fichiers sont alors impliqués (le fichier de configuration et le contrôleur PHP). Avec les annotations, la configuration se fait directement dans le contrôleur.

Pour pouvoir utiliser les annotations, nous devons ajouter une autre dépendance :

.. code-block:: bash

    $ symfony composer req annotations

Vous vous demandez peut-être comment vous pouvez deviner le nom du paquet à installer pour une fonctionnalité donnée ? La plupart du temps, vous n'avez pas besoin de le savoir, car Symfony propose le nom du paquet à installer dans ses messages d'erreur. Par exemple, exécuter ``symfony make:controller`` sans le paquet ``annotations`` se terminerait par une exception contenant une indication sur le bon paquet à installer.

Générer un contrôleur
------------------------

.. index::
    single: Command;make:controller

Créez votre premier *Controller* avec la commande ``make:controller`` :

.. code-block:: bash

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Annotations;Route

La commande crée une classe ``ConferenceController`` dans le répertoire  ``src/Controller/``. La classe générée contient du code standard prêt à être ajusté :

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Annotation\Route;

    class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

L'annotation ``#[Route('/conference', name: 'conference')]`` est ce qui fait de la méthode ``index()`` un contrôleur (la configuration est à côté du code qu'elle configure).

Lorsque vous visitez la page ``/conference`` dans un navigateur, le contrôleur est exécuté et une réponse est renvoyée.

Modifiez la route afin qu'elle corresponde à la page d'accueil :

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

Le nom de la route (``name``) sera utile lorsque nous voudrons faire référence à la page d'accueil dans notre code. Au lieu de coder en dur le chemin ``/``, nous utiliserons le nom de la route.

À la place de la page par défaut, retournons une simple page HTML :

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +<html>
    +    <body>
    +        <img src="/images/under-construction.gif" />
    +    </body>
    +</html>
    +EOF
    +        );
         }
     }

Rafraîchissez le navigateur :

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

La responsabilité principale d'un contrôleur est de retourner une réponse HTTP (``Response``) pour la requête.

.. _easter-egg:

Ajouter un *easter egg*
-----------------------

Pour montrer comment une réponse peut tirer parti de l'information contenue dans la requête, ajoutons un petit `easter egg`_. Lorsqu'une requête vers la page d'accueil sera réalisée avec un paramètre d'URL comme ``?hello=Fabien``, nous ajouterons du texte pour saluer la personne :

.. code-block:: diff
    :emphasize-lines: 16
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -3,6 +3,7 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;

    @@ -11,11 +12,17 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

Symfony expose les données de la requête à travers un objet ``Request``. Lorsque Symfony voit un argument de contrôleur avec ce typage précis, il sait automatiquement qu'il doit vous le passer. Nous pouvons l'utiliser pour récupérer le nom depuis le paramètre d'URL et ajouter un titre ``<h1>``.

Dans un navigateur, rendez-vous sur ``/``, puis sur ``/?hello=Fabien`` pour constater la différence.

.. note::

    Remarquez l'appel à ``htmlspecialchars()``, pour éviter les attaques XSS. Ce sera fait automatiquement pour nous lorsque nous passerons à un moteur de template digne de ce nom.

Nous aurions également pu inclure le nom directement dans l'URL :

.. code-block:: diff
    :emphasize-lines: 10,11
    :class: ignore

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,13 +9,19 @@ use Symfony\Component\Routing\Annotation\Route;
     class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    +    #[Route('/hello/{name}', name: 'homepage')]
    -    public function index(): Response
    +    public function index(string $name = ''): Response
         {
    +        $greet = '';
    +        if ($name) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
     <html>
         <body>
    +        $greet
             <img src="/images/under-construction.gif" />
         </body>
     </html>

La partie de la route ``{name}`` est un *paramètre de route* dynamique - il fonctionne comme un joker. Vous pouvez maintenant vous rendre sur ``/hello`` et sur ``/hello/Fabien`` dans un navigateur pour obtenir les mêmes résultats qu'auparavant. Vous pouvez récupérer la *valeur* du paramètre ``{name}`` en ajoutant un argument portant le même *nom* au contrôleur, donc ``$name``.

.. sidebar:: Aller plus loin

    * Le système de `routage <https://symfony.com/doc/current/routing.html>`_ de Symfony ;

    * `SymfonyCasts : tutoriels sur les routes, contrôleurs et pages <https://symfonycasts.com/screencast/symfony/route-controller>`_ ;

    * `Annotations <https://www.doctrine-project.org/projects/doctrine-annotations/en/1.6/annotations.html>`_ ; en PHP ;

    * Le composant `HttpFoundation <https://symfony.com/doc/current/components/http_foundation.html>`_ ;

    * Attaques de sécurité `XSS (Cross-Site Scripting) <https://owasp.org/www-community/attacks/xss/>`_ ;

    * La `cheat sheet du système de routage Symfony <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf>`_.

.. _`easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
