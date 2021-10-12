Impostare un pannello amministrativo
====================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Aggiungere le prossime conferenze al database è compito degli amministratori del progetto. Il pannello amministrativo è una sezione protetta del sito web dove gli *amministratori di progetto* possono gestire i dati del sito, moderare i feedback e altro ancora.

Come possiamo crearlo in fretta? Usando un bundle che è in grado di generare un pannello amministrativo basato sul modello del progetto. EasyAdmin si adatta perfettamente al contesto.

Configurare EasyAdmin
---------------------

Per iniziare, aggiungiamo EasyAdmin come dipendenza del progetto:

.. code-block:: bash

    $ symfony composer req "admin:^2"

Per configurare EasyAdmin, è stato generato un nuovo file di configurazione, tramite la sua ricetta Flex:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

Quasi tutti i pacchetti installati hanno una configurazione come questa nel percorso ``config/packages/``. Nella maggior parte dei casi, le impostazioni predefinite sono state scelte attentamente per funzionare per gran parte delle applicazioni.

Rimuovere i commenti dalle prime due righe e aggiungere le classi del modello del progetto:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

Accedere al pannello amministrativo generato in ``/admin``. Boom! Un'interfaccia amministrativa bella e ricca di funzionalità per conferenze e commenti:

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

Fare lo stesso per la classe ``Comment``:

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

Ora è possibile aggiungere/modificare/cancellare le conferenze direttamente dal pannello amministrativo. Si può fare qualche prova e aggiungere almeno una conferenza.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

Aggiungiamo qualche commento senza foto e impostiamo la data manualmente: la colonna ``createdAt`` sarà automatizzata in un secondo momento.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

Personalizzazione di EasyAdmin
------------------------------

Il pannello amministrativo predefinito funziona bene, ma può essere personalizzato in molti modi per migliorare l'esperienza utente. Facciamo alcune semplici modifiche per dimostrarne le possibilità. Modificare la configurazione corrente con quanto segue:

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

Queste personalizzazioni sono solo una piccola introduzione alle possibilità offerte da EasyAdmin.

Giocate con il pannello amministrativo, filtrando i commenti per conferenza o cercando i commenti per e-mail, ad esempio. L'unico problema è che chiunque può accedere al backend. Niente paura, l'accesso sarà regolato in un secondo momento.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Andare oltre

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Riferimento alla configurazione del framework Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_.
