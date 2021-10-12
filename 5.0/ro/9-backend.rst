Configurarea unei interfețe de administrare
============================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Adăugarea conferințelor viitoare în baza de date va fi sarcina administratorilor de proiect. O *interfață de administrare* este o secțiune protejată a site-ului web în care *administratorii* pot gestiona datele site-ului, pot modera recenziile și multe altele.

Cum o putem crea rapid? Folosind un pachet care este capabil să genereze partea de administrare după modelul proiectului. EasyAdmin se potrivește perfect.

Configurarea EasyAdmin
----------------------

Mai întâi, adaugă EasyAdmin ca dependență de proiect:

.. code-block:: bash

    $ symfony composer req "admin:^2"

Pentru a configura EasyAdmin, a fost generat un nou fișier de configurare prin rețeta sa Flex:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

Aproape toate pachetele instalate au o configurație ca aceasta în directorul ``config/packages/``. De cele mai multe ori, valorile implicite au fost alese cu atenție pentru a funcționa la majoritatea aplicațiilor.

Decomentează primele două linii și adaugă clasele modelului din proiect:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

Accesează interfața generată la ``/admin``. Boom! O interfață de administrare plăcută și bogată în funcționalități pentru conferințe și comentarii:

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

La fel și pentru clasa ``Comment``:

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

Acum poți adăuga, modifica sau șterge conferințe direct din interfața de administrare. Testează-l adăugând cel puțin o conferință.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

Adaugă și câteva comentarii fără fotografii. Setează manual data creării, pentru moment; vom completa automat coloana ``createdAt`` într-o etapă ulterioară.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

Personalizarea EasyAdmin
------------------------

Interfața de administrare funcționează bine, dar poate fi personalizată pentru a îmbunătăți experiența utilizatorului. Să facem câteva modificări simple pentru a demonstra ce putem schimba. Înlocuiește configurația curentă cu următoarele:

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

Aceste personalizări sunt doar o mică introducere a posibilităților oferite de EasyAdmin.

Joacă-te cu interfața. De exemplu, filtrează comentariile după conferință sau caută comentarii semnate cu o anumită adresă de email. Singura problemă este că oricine poate avea acces la backend. Nu-ți face griji, îl vom restricționa imediat.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Mergând mai departe

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Referință de configurare a framework-ului Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
