Konfigurowanie panelu administracyjnego
=======================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Dodawanie nadchodzących konferencji do bazy danych jest zadaniem administratorów i administratorek projektu. *Panel administracyjny* to chroniona część strony internetowej, w której *osoby korzystające z konta administracyjnego* mogą zarządzać danymi strony internetowej, moderować przesłane opinie i wiele innych.

Jak możemy to szybko stworzyć? Za pomocą pakietu, który jest w stanie wygenerować panel administracyjny w oparciu o model projektu. EasyAdmin idealnie się do tego nadaje.

Konfigurowanie EasyAdmin
------------------------

Po pierwsze, dodaj EasyAdmin jako zależność do projektu:

.. code-block:: bash

    $ symfony composer req "admin:^2"

Za pomocą przepisu (ang. recipe) Flex wygenerowany został nowy plik konfiguracyjny, umożliwiający konfigurację biblioteki EasyAdmin.

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

Prawie wszystkie zainstalowane pakiety mają taką konfigurację w katalogu ``config/packages/``. W większości przypadków domyślne ustawienia zostały starannie dobrane tak, aby działały w większości aplikacji.

Odkomentuj kilka pierwszych linii i dodaj klasy modeli z projektu:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

Wejdź do wygenerowanego panelu administracyjnego odwiedzając ``/admin``. Bum! Oto ładny i bogaty w funkcje interfejs administracyjny dla konferencji i komentarzy:

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

Zrób to samo dla klasy ``Comment``:

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

Teraz możesz dodawać/modyfikować/usuwać konferencje bezpośrednio z panelu administracyjnego. Pobaw się nim i dodaj co najmniej jedną konferencję.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

Dodaj kilka komentarzy bez zdjęć. Na razie ustaw datę ręcznie; kolumnę ``createdAt`` wypełnimy automatycznie w późniejszym kroku.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

Dostosowywanie EasyAdmin
------------------------

Domyślny panel administracyjny działa dobrze, ale można go dostosować na wiele sposobów, aby usprawnić jego działanie. Zróbmy kilka prostych zmian, aby zademonstrować jego możliwości. Zastąp bieżącą konfigurację poniższą:

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

Te modyfikacje są tylko małą prezentacją możliwości, jakie daje nam EasyAdmin.

Pobaw się panelem administracyjnym, wyfiltruj komentarze po konferencji lub na przykład wyszukuj je po adresie e-mail. Jedynym problemem jest to, że każdy ma do niego dostęp. Nie martw się, zabezpieczymy go w przyszłości.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Idąc dalej

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Dostępne ustawienia w konfiguracji Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
