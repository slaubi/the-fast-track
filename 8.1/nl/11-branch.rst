Branching van de code
=====================

Er zijn vele manieren om de workflow van codewijzigingen in een project te organiseren. Maar direct werken in de Git master-branch en direct in productie nemen zonder testen is waarschijnlijk niet de beste.

Testen gaat niet alleen over unit- of functionele testen, het gaat ook over het controleren hoe de applicatie zich gedraagt met productiegegevens. Als jij of jouw `stakeholders`_ de applicatie kunnen testen in dezelfde staat als aan de eindgebruikers opgeleverd zal worden, dan is dat een groot voordeel. Het maakt dat je met vertrouwen kan deployen. Het voordeel komt in het bijzonder tot zijn recht wanneer niet-technische mensen nieuwe features kunnen gaan valideren.

Om het eenvoudig te houden zullen we in de volgende stappen al ons werk in de Git master branch blijven doen. Maar laten we eens kijken hoe het eventueel ook beter kan.

Het eigen maken van een Git Workflow
------------------------------------

Een mogelijke workflow is het creëren van een branch per nieuwe functie of bugfix. Dit is eenvoudig en efficiënt.

Branches maken
--------------

.. index::
    single: Git;branch
    single: Git;checkout

De workflow begint met het aanmaken van een Git branch:

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

Dit commando maakt een ``sessions-in-db`` branch aan vanuit de ``master`` branch. Het maakt een afsplitsing van de code en de infrastructuurconfiguratie.

Sessies opslaan in de database
------------------------------

.. index::
    single: Session;Database

Zoals je misschien - op basis van de branchnaam - geraden hebt, willen we de session-storage overschakelen van het filesystem naar een database (in dit geval onze PostgreSQL database).

De stappen die nodig zijn om het te realiseren zijn logisch:

#. Maak een Git branch aan;

#. Werk de Symfony-configuratie bij indien nodig;

#. Schrijf en/of update wat code indien nodig;

#. Update the PHP configuration if needed (like adding the PostgreSQL PHP
   extension);

#. Update de infrastructuur op Docker en Platform.sh indien nodig (voeg bijvoorbeeld de PostgreSQL-service toe);

#. Lokaal testen;

#. Remote testen;

#. Merge de branch met de master branch;

#. Deployen naar productie;

#. Verwijder de branch.

Verander de ``session.handler_id`` configuratie en verwijs naar de database DSN om sessies in de database op te slaan:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -8,7 +8,7 @@ framework:
         # Enables session support. Note that the session will ONLY be started if you read or write from it.
         # Remove or comment this section to explicitly disable session support.
         session:
    -        handler_id: null
    +        handler_id: '%env(resolve:DATABASE_URL)%'
             cookie_secure: auto
             cookie_samesite: lax
             storage_factory_id: session.storage.factory.native

We moeten een ``sessions`` tabel toevoegen om de gegevens in de database op te slaan. Dit doen we middels een Doctrine migratie:

.. code-block:: terminal

    $ symfony console make:migration

Wijzig het bestand en voeg de code toe om de database tabel te creëren in de ``up()`` methode:

.. code-block:: diff
    :caption: patch_file

    --- a/migrations/Version00000000000000.php
    +++ b/migrations/Version00000000000000.php
    @@ -21,6 +21,15 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs

    +        $this->addSql('
    +            CREATE TABLE sessions (
    +                sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    +                sess_data BYTEA NOT NULL,
    +                sess_lifetime INTEGER NOT NULL,
    +                sess_time INTEGER NOT NULL
    +            )
    +        ');
    +        $this->addSql('CREATE INDEX expiry ON sessions (sess_lifetime)');
         }

         public function down(Schema $schema): void

Migreer de database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Test lokaal door naar de website te surfen. Omdat er geen visuele veranderingen zijn en omdat we nog geen sessies gebruiken, zou alles nog steeds moeten werken zoals voorheen.

.. note::

    We hebben stappen 3 t/m 5 hier niet nodig omdat we de database hergebruiken als session-storage. Het hoofdstuk over Redis laat zien hoe eenvoudig het is om nieuwe services toe te voegen, te testen en te deployen in zowel Docker als Platform.sh.

Omdat de nieuwe tabel niet "gemanaged" is door Docker, zullen we in de Docker migratie aan moeten geven dat deze niet verwijderd moet worden in de volgende database migratie:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/doctrine.yaml
    +++ b/config/packages/doctrine.yaml
    @@ -5,6 +5,8 @@ doctrine:
             # IMPORTANT: You MUST configure your server version,
             # either here or in the DATABASE_URL env var (see .env file)
             #server_version: '14'
    +
    +        schema_filter: ~^(?!session)~
         orm:
             auto_generate_proxy_classes: true
             naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware

Commit je wijzigingen in de nieuwe branch:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Deployen van een branch
-----------------------

.. index::
    single: Platform.sh;Environment

Voordat we de branch in productie nemen, moeten we deze testen op dezelfde infrastructuur als in productie. We moeten ook valideren dat alles goed werkt voor de ``prod`` Symfony-omgeving (de lokale website gebruikt de ``dev`` Symfony-omgeving).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Laten we nu een *Platform.sh-omgeving* maken op basis van de *Git branch*:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:deploy

Dit commando creëert vervolgens een nieuwe omgeving:

* De branch erft de code en infrastructuur van de huidige Git branch (``sessions-in-db``);

* De gegevens komen uit de master- (aka-productie) omgeving door het maken van een consistente snapshot van alle service-gegevens, inclusief bestanden (bijvoorbeeld door de gebruiker geüploade bestanden) en databases;

* Er wordt een nieuw dedicated cluster gecreëerd om de code, de gegevens en de infrastructuur te deployen.

Aangezien de deployment dezelfde stappen volgt als de deployment naar productie, zullen ook databasemigraties worden uitgevoerd. Dit is een goede manier om te valideren dat de migraties werken met de productiedataset.

De niet-``master`` omgevingen lijken erg op de ``master`` omgeving, op enkele kleine verschillen na: e-mails worden bijvoorbeeld standaard niet verstuurd.

.. index::
    single: Symfony CLI;cloud:url

Zodra de deployment is voltooid, open je de nieuwe branch in een browser:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Merk op dat alle Platform.sh commando's werken op de huidige Git branch. Dit commando opent de gedeployede URL voor de ``sessions-in-db`` branch; de URL ziet er dan als volgt uit ``https://sessions-in-db-xxx.eu-5.platformsh.site/``.

Test de website op deze nieuwe omgeving, je zou alle gegevens moeten zien die je in de master-omgeving hebt aangemaakt.

Als je meer conferenties aan de ``master`` omgeving toevoegt, dan zullen ze niet in de ``sessions-in-db`` omgeving verschijnen en vice versa. De omgevingen zijn onafhankelijk en geïsoleerd.

Als de code op master evolueert, kun je altijd de Git branch rebasen en de bijgewerkte versie deployen, waardoor de conflicten voor zowel de code als de infrastructuur worden opgelost.

.. index::
    single: Symfony CLI;cloud:env:sync

Je kan zelfs de gegevens van de master terug naar de ``sessions-in-db`` omgeving synchroniseren:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Het debuggen van productie-deployment vóór de ingebruikname
-------------------------------------------------------------

.. index::
    single: Platform.sh;Debugging

Standaard gebruiken alle Platform.sh-omgevingen dezelfde instellingen als de ``master`` / ``prod`` omgeving (ook wel de Symfony ``prod`` omgeving genoemd). Dit stelt je in staat om de toepassing in reële omstandigheden te testen. Het geeft je het gevoel dat je direct op productieservers ontwikkelt en test, maar zonder de risico's die daarmee gepaard gaan. Dit doet me denken aan de goede oude tijd toen we deployments via FTP verzorgden.

.. index::
    single: Symfony CLI;cloud:env:debug

In het geval van een probleem, kan je misschien overstappen naar de Symfony ``dev`` omgeving:

.. code-block:: terminal

    $ symfony cloud:env:debug

Als je klaar bent, ga dan terug naar de productie-instellingen:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    Schakel **nooit** de ``dev`` omgeving of de Symfony Profiler in op de ``master`` branch; het maakt de applicatie traag en zorgt voor een hoop serieuze kwetsbaarheden.

Testen van de productie-deployments voor de ingebruikname
---------------------------------------------------------

Toegang hebben tot de toekomstige versie van de website met productiegegevens biedt veel mogelijkheden: van visuele regressietests tot performance tests. `Blackfire`_ is de perfecte tool voor de klus.

Raadpleeg de stap over :doc:`Prestaties <29-performance>` om meer te weten te komen over hoe je Blackfire kan gebruiken om jouw code te testen voordat je deze deployt.

Mergen naar productie
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Als je tevreden bent met de wijzigingen in de branch, merge dan de code en de infrastructuur naar de Git master branch:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

En deploy:

.. code-block:: terminal

    $ symfony cloud:deploy

Bij de deployment worden enkel de code en wijzigingen in de infrastructuur naar Platform.sh gepusht; de data wordt op geen enkele wijze beïnvloed.

Opruimen
--------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Ruim tot slot op door de Git branch en de Platform.sh omgeving te verwijderen:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Verder gaan

    * `Git branching`_;

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Git branching`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
