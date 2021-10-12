Een admin-backend opzetten
==========================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Het toevoegen van aankomende conferenties aan de database is de taak van de projectbeheerders. Een *admin-backend* is een beschermd deel van de website waar *projectbeheerders* de gegevens van de website kunnen beheren, feedback kunnen modereren, en meer.

Hoe kunnen we dit snel creëren? Door gebruik te maken van een bundle die in staat is om een admin-backend te genereren op basis van het model van het project. EasyAdmin past perfect in dit plaatje.

EasyAdmin configureren
----------------------

Voeg eerst EasyAdmin toe als projectdependency:

.. code-block:: bash

    $ symfony composer req "admin:^2"

Door het Flex recipe is een nieuw configuratiebestand gemaakt, zodat EasyAdmin geconfigureerd kan worden:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

Bijna alle geïnstalleerde packages hebben een configuratie zoals deze in de ``config/packages/``-map. Meestal zijn de standaardinstellingen zorgvuldig gekozen om voor de meeste toepassingen te werken.

Haal de eerste paar regels uit het commentaar en voeg de model classes van het project toe:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

Ga naar de gegenereerde admin-backend op ``/admin`` . Boem! Een mooie en veelzijdige admin-interface voor conferenties en reacties:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

.. tip::

    Why is the backend accessible under ``/admin``? That's the default prefix configured in ``config/routes/easy_admin.yaml``:

    .. code-block:: yaml
        :caption: config/routes/easy_admin.yaml
        :class: ignore

        easy_admin_bundle:
            resource: '@EasyAdminBundle/Controller/EasyAdminController.php'
            prefix: /admin
            type: annotation

    You can change it to anything you like.

Adding conferences and comments is not possible yet as you would get an error: ``Object of class App\Entity\Conference could not be converted to string``. EasyAdmin tries to display the conference related to comments, but it can only do so if there is a string representation of a conference. Fix it by adding a ``__toString()`` method on the ``Conference`` class:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -44,6 +44,11 @@ class Conference
             $this->comments = new ArrayCollection();
         }

    +    public function __toString(): string
    +    {
    +        return $this->city.' '.$this->year;
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

Doe hetzelfde voor de ``Comment``-class:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -48,6 +48,11 @@ class Comment
          */
         private $photoFilename;

    +    public function __toString(): string
    +    {
    +        return (string) $this->getEmail();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

Je kan nu rechtstreeks vanuit de admin-backend conferenties toevoegen, wijzigen en verwijderen. Speel ermee en voeg ten minste één conferentie toe.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

Voeg een aantal reacties zonder foto's toe. Stel de datum voorlopig handmatig in; we vullen de ``createdAt`` kolom in een later stadium automatisch in.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

EasyAdmin aanpassen
-------------------

De standaard admin-backend werkt goed, maar kan op vele manieren worden aangepast om de gebruikerservaring te verbeteren. Laten we enkele eenvoudige aanpassingen doen om de mogelijkheden te demonstreren. Vervang de huidige configuratie door het volgende:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        site_name: Conference Guestbook

        design:
            menu:
                - { route: 'homepage', label: 'Back to the website', icon: 'home' }
                - { entity: 'Conference', label: 'Conferences', icon: 'map-marker' }
                - { entity: 'Comment', label: 'Comments', icon: 'comments' }

        entities:
            Conference:
                class: App\Entity\Conference

            Comment:
                class: App\Entity\Comment
                list:
                    fields:
                        - author
                        - { property: 'email', type: 'email' }
                        - { property: 'createdAt', type: 'datetime' }
                    sort: ['createdAt', 'ASC']
                    filters: ['conference']
                edit:
                    fields:
                        - { property: 'conference' }
                        - { property: 'createdAt', type: datetime, type_options: { disabled: true } }
                        - 'author'
                        - { property: 'email', type: 'email' }
                        - text

We have overridden the ``design`` section to add icons to the menu items and to add a link back to the website home page.

For the ``Comment`` section, listing the fields lets us order them the way we want. Some fields are tweaked, like setting the creation date to read-only. The ``filters`` section defines which filters to expose on top of the regular search field.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

Deze aanpassingen zijn slechts een kleine introductie van de mogelijkheden van EasyAdmin.

Speel met de admin, filter de reacties per conferentie, of zoek bijvoorbeeld op basis van het e-mailadres naar reacties. Het enige probleem is dat iedereen toegang heeft tot de backend. Maak je geen zorgen, we zullen dit in de volgende stap veilig maken.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Verder gaan

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Configuratiehandleiding voor het Symfony framework <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
