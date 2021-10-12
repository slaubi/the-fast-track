Configurando un panel de administración
========================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

Añadir las próximas conferencias a la base de datos es tarea de los administradores del proyecto. Un *panel de administración* es una sección protegida del sitio web donde *los administradores del proyecto* pueden  gestionar los datos del sitio web, moderar los comentarios y efectuar otro tipo de operaciones.

¿Cómo podemos crearlo rápidamente? Utilizando un *bundle* que  genera un panel de administración a partir del modelo del proyecto. EasyAdmin es perfecto para esta tarea.

Configurando EasyAdmin
----------------------

En primer lugar, añade EasyAdmin como una dependencia del proyecto:

.. code-block:: bash

    $ symfony composer req "admin:^2"

Para configurar EasyAdmin, se ha generado un nuevo archivo de configuración a través de su receta de Flex:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml
    :class: ignore

    #easy_admin:
    #    entities:
    #        # List the entity class name you want to manage
    #        - App\Entity\Product
    #        - App\Entity\Category
    #        - App\Entity\User

La práctica totalidad de los paquetes instalados tienen una configuración similar a ésta en el directorio ``config/packages/``. Casi siempre, los valores predeterminados habrán sido escogidos cuidadosamente para que funcionen en la mayoría de las aplicaciones.

Quita la marca de comentarios del primer par de líneas y agrega las clases del modelo del proyecto:

.. code-block:: yaml
    :caption: config/packages/easy_admin.yaml

    easy_admin:
        entities:
            - App\Entity\Conference
            - App\Entity\Comment

Accede al panel de administración generado en ``/admin``. ¡Boom! Ya dispones de una interfaz de administración para conferencias y comentarios agradable y rica en funciones:

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

Haz lo mismo para la clase ``Comment``:

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

Ahora puedes añadir/modificar/eliminar conferencias directamente desde el panel de administración. Juega con él y añade al menos una conferencia.

.. figure:: screenshots/easy-admin.png
    :alt: /admin/?entity=Conference&action=list
    :align: center
    :figclass: with-browser

Añade algunos comentarios sin fotos. De momento, configura la fecha manualmente; rellenaremos la columna ``createdAt`` automáticamente en un paso posterior.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin/?entity=Comment&action=list
    :align: center
    :figclass: with-browser

Personalizando EasyAdmin
------------------------

El panel de administración por defecto funciona bien, pero se puede personalizar de muchas maneras para mejorar la experiencia. Hagamos algunos cambios sencillos para demostrar sus posibilidades. Sustituye la configuración actual por la siguiente:

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

Estas personalizaciones son solo una pequeña introducción a las posibilidades que ofrece EasyAdmin.

Juega con el panel, filtra los comentarios por conferencia, o busca comentarios por correo electrónico, por ejemplo. Pero todavía nos queda algo por resolver: cualquiera puede acceder al panel de administración. No te preocupes, lo protegeremos más adelante.

.. code-block:: bash
    :class: hide

    $ symfony run psql -c "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: Yendo más allá

    * `EasyAdmin docs <https://symfony.com/doc/2.x/bundles/EasyAdminBundle/index.html>`_;

    * `SymfonyCasts EasyAdminBundle tutorial <https://symfonycasts.com/screencast/easyadminbundle>`_;

    * `Configuración de referencia del *framework* Symfony <https://symfony.com/doc/current/reference/configuration/framework.html>`_ .
