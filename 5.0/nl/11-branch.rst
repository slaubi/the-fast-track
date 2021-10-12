Branching van de code
=====================

Er zijn vele manieren om de workflow van codewijzigingen in een project te organiseren. Maar direct werken in de Git master-branch en direct in productie nemen zonder testen is waarschijnlijk niet de beste.

Testen gaat niet alleen over unit- of functionele testen, het gaat ook over het controleren hoe de applicatie zich gedraagt met productiegegevens. Als jij of jouw `stakeholders`_ de applicatie kunnen testen in dezelfde staat als aan de eindgebruikers opgeleverd zal worden, dan is dat een groot voordeel. Het maakt dat je met vertrouwen kan deployen. Het voordeel komt in het bijzonder tot zijn recht wanneer niet-technische mensen nieuwe features kunnen gaan valideren.

Om het eenvoudig te houden zullen we in de volgende stappen al ons werk in de Git master branch blijven doen. Maar laten we eens kijken hoe het eventueel ook beter kan.

Het eigen maken van een Git Workflow
------------------------------------

Een mogelijke workflow is het creëren van een branch per nieuwe functie of bugfix. Dit is eenvoudig en efficiënt.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Branches maken
--------------

.. index::
    single: Git;branch
    single: Git;checkout

De workflow begint met het aanmaken van een Git branch:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

Dit commando maakt een ``sessions-in-redis`` branch aan vanuit de ``master`` branch. Het maakt een afsplitsing van de code en de infrastructuurconfiguratie.

Sessies opslaan in Redis
------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

Zoals je misschien - op basis van de branchnaam - geraden hebt, willen we de session-storage overschakelen van het filesystem naar Redis.

De stappen die nodig zijn om het te realiseren zijn logisch:

#. Maak een Git branch aan;

#. Werk de Symfony-configuratie bij indien nodig;

#. Schrijf en/of update wat code indien nodig;

#. Werk de PHP-configuratie bij (voeg de Redis PHP-extensie toe);

#. Update de infrastructuur op Docker en SymfonyCloud (voeg de Redis-service toe);

#. Lokaal testen;

#. Remote testen;

#. Merge de branch met de master branch;

#. Deployen naar productie;

#. Verwijder de branch.

All changes needed for 2 to 5 can be done in one patch:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony.cloud.yaml
    +++ b/.symfony.cloud.yaml
    @@ -4,6 +4,7 @@ type: php:7.4

     runtime:
         extensions:
    +        - redis
             - pdo_pgsql
             - apcu
             - mbstring
    @@ -24,6 +25,7 @@ disk: 512

     relationships:
         database: "db:postgresql"
    +    redis: "rediscache:redis"

     web:
         locations:
    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,6 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +rediscache:
    +    type: redis:5.0
    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -7,7 +7,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(REDIS_URL)%'
             cookie_secure: auto
             cookie_samesite: lax

    --- a/docker-compose.yaml
    +++ b/docker-compose.yaml
    @@ -8,3 +8,7 @@ services:
                 POSTGRES_PASSWORD: main
                 POSTGRES_DB: main
             ports: [5432]
    +
    +    redis:
    +        image: redis:5-alpine
    +        ports: [6379]

Isn't it *beautiful*?

"Reboot" Docker to start the Redis service:

.. code-block:: bash

    $ docker-compose stop
    $ docker-compose up -d

Ik laat je lokaal testen door naar de website te surfen. Omdat er geen visuele veranderingen zijn en omdat we nog geen sessies gebruiken, zou alles nog steeds moeten werken zoals voorheen.

Deployen van een branch
-----------------------

.. index::
    single: SymfonyCloud;Environment

Voordat we de branch in productie nemen, moeten we deze testen op dezelfde infrastructuur als in productie. We moeten ook valideren dat alles goed werkt voor de ``prod`` Symfony-omgeving (de lokale website gebruikt de ``dev`` Symfony-omgeving).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Laten we nu een *SymfonyCloud-omgeving* maken op basis van de *Git branch*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

Dit commando creëert vervolgens een nieuwe omgeving:

* De branch erft de code en infrastructuur van de huidige Git branch (``sessions-in-redis``);

* De gegevens komen uit de master- (aka-productie) omgeving door het maken van een consistente snapshot van alle service-gegevens, inclusief bestanden (bijvoorbeeld door de gebruiker geüploade bestanden) en databases;

* Er wordt een nieuw dedicated cluster gecreëerd om de code, de gegevens en de infrastructuur te deployen.

Aangezien de deployment dezelfde stappen volgt als de deployment naar productie, zullen ook databasemigraties worden uitgevoerd. Dit is een goede manier om te valideren dat de migraties werken met de productiedataset.

De niet-``master`` omgevingen lijken erg op de ``master`` omgeving, op enkele kleine verschillen na: e-mails worden bijvoorbeeld standaard niet verstuurd.

.. index::
    single: Symfony CLI;open:remote

Zodra de deployment is voltooid, open je de nieuwe branch in een browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Merk op dat alle SymfonyCloud commando's werken op de huidige Git branch. Dit commando opent de deployede URL voor de ``sessions-in-redis`` branch; de URL ziet er dan als volgt uit ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Test de website op deze nieuwe omgeving, je zou alle gegevens moeten zien die je in de master-omgeving hebt aangemaakt.

Als je meer conferenties aan de ``master`` omgeving toevoegt, dan zullen ze niet in de ``sessions-in-redis`` omgeving verschijnen en vice versa. De omgevingen zijn onafhankelijk en geïsoleerd.

Als de code op master evolueert, kun je altijd de Git branch rebasen en de bijgewerkte versie deployen, waardoor de conflicten voor zowel de code als de infrastructuur worden opgelost.

.. index::
    single: Symfony CLI;env:sync

Je kan zelfs de gegevens van de master terug naar de ``sessions-in-redis`` omgeving synchroniseren:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Het debuggen van productie-deployment vóór de ingebruikname
-------------------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

Standaard gebruiken alle SymfonyCloud-omgevingen dezelfde instellingen als de ``master`` / ``prod`` omgeving (ook wel de Symfony ``prod`` omgeving genoemd). Dit stelt je in staat om de toepassing in reële omstandigheden te testen. Het geeft je het gevoel dat je direct op productieservers ontwikkelt en test, maar zonder de risico's die daarmee gepaard gaan. Dit doet me denken aan de goede oude tijd toen we deployments via FTP verzorgden.

.. index::
    single: Symfony CLI;env:debug

In het geval van een probleem, kan je misschien overstappen naar de Symfony ``dev`` omgeving:

.. code-block:: bash

    $ symfony env:debug

Als je klaar bent, ga dan terug naar de productie-instellingen:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    Schakel **nooit** de ``dev`` omgeving of de Symfony Profiler in op de ``master`` branch; het maakt de applicatie traag en zorgt voor een hoop serieuze kwetsbaarheden.

Testen van de productie-deployments voor de ingebruikname
---------------------------------------------------------

Toegang hebben tot de nieuwe versie van de website met productiegegevens biedt veel mogelijkheden: van visuele regressietests tot performance tests. `Blackfire <https://blackfire.io>`_ is de perfecte tool voor de klus.

Raadpleeg de stap over "Prestaties" om meer te weten te komen over hoe je Blackfire kan gebruiken om jouw code te testen voordat je deze deployt.

Mergen naar productie
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Als je tevreden bent met de wijzigingen in de branch, merge dan de code en de infrastructuur naar de Git master branch:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

En deploy:

.. code-block:: bash

    $ symfony deploy

Bij de deployment worden enkel de code en wijzigingen in de infrastructuur naar SymfonyCloud gepusht; de data wordt op geen enkele wijze beïnvloed.

Opruimen
--------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Ruim tot slot op door de Git branch en de SymfonyCloud omgeving te verwijderen:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Verder gaan

    * `Git branching <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
