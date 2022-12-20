Branching the Code
==================

There are many ways to organize the workflow of code changes in a project. But working directly on the Git master branch and deploying directly to production without testing is probably not the best one.

Testing is not just about unit or functional tests, it is also about checking the application behavior with production data. If you or your `stakeholders`_ can browse the application exactly as it will be deployed to end users, this becomes a huge advantage and allows you to deploy with confidence. It is especially powerful when non-technical people can validate new features.

We will continue doing all the work in the Git master branch in the next steps for simplicity sake and to avoid repeating ourselves, but let's see how this could work better.

Adopting a Git Workflow
-----------------------

One possible workflow is to create one branch per new feature or bug fix. It is simple and efficient.

Creating Branches
-----------------

.. index::
    single: Git;branch
    single: Git;checkout

The workflow starts with the creation of a Git branch:

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

This command creates a ``sessions-in-db`` branch from the ``master`` branch. It "forks" the code and the infrastructure configuration.

Storing Sessions in the Database
--------------------------------

.. index::
    single: Session;Database

As you might have guessed from the branch name, we want to switch session storage from the filesystem to a database store (our PostgreSQL database here).

The needed steps to make it a reality are typical:

#. Create a Git branch;

#. Update the Symfony configuration if needed;

#. Write and/or update some code if needed;

#. Update the PHP configuration if needed (like adding the PostgreSQL PHP
   extension);

#. Update the infrastructure on Docker and Platform.sh if needed (add the PostgreSQL service);

#. Test locally;

#. Test remotely;

#. Merge the branch to master;

#. Deploy to production;

#. Delete the branch.

To store sessions in the database, change the ``session.handler_id`` configuration to point to the database DSN:

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

To store sessions in the database, we need to create the ``sessions`` table. Do so with a Doctrine migration:

.. code-block:: terminal

    $ symfony console make:migration

Edit the file to add the table creation in the ``up()`` method:

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

Migrate the database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Test locally by browsing the website. As there are no visual changes and because we are not using sessions yet, everything should still work as before.

.. note::

    We don't need steps 3 to 5 here as we are re-using the database as the session storage, but the chapter about using Redis shows how straightforward it is to add, test, and deploy a new service in both Docker and Platform.sh.

As the new table is not "managed" by Doctrine, we must configure Doctrine to not remove it in the next database migration:

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

Commit your changes to the new branch:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Deploying a Branch
------------------

.. index::
    single: Platform.sh;Environment

Before deploying to production, we should test the branch on the same infrastructure as the production one. We should also validate that everything works fine for the Symfony ``prod`` environment (the local website used the Symfony ``dev`` environment).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Now, let's create a *Platform.sh environment* based on the *Git branch*:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:deploy

This command creates a new environment as follows:

* The branch inherits the code and infrastructure from the current Git branch (``sessions-in-db``);

* The data come from the master (aka production) environment by taking a consistent snapshot of all service data, including files (user uploaded files for instance) and databases;

* A new dedicated cluster is created to deploy the code, the data, and the infrastructure.

As the deployment follows the same steps as deploying to production, database migrations will also be executed. This is a great way to validate that the migrations work with production data.

The non-``master`` environments are very similar to the ``master`` one except for some small differences: for instance, emails are not sent by default.

.. index::
    single: Symfony CLI;cloud:url

Once the deployment is finished, open the new branch in a browser:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Note that all Platform.sh commands work on the current Git branch. This command opens the deployed URL for the ``sessions-in-db`` branch; the URL will look like ``https://sessions-in-db-xxx.eu-5.platformsh.site/``.

Test the website on this new environment, you should see all the data that you created in the master environment.

If you add more conferences on the ``master`` environment, they won't show up in the ``sessions-in-db`` environment and vice-versa. The environments are independent and isolated.

If the code evolves on master, you can always rebase the Git branch and deploy the updated version, resolving the conflicts for both the code and the infrastructure.

.. index::
    single: Symfony CLI;cloud:env:sync

You can even synchronize the data from master back to the ``sessions-in-db`` environment:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Debugging Production Deployments before Deploying
-------------------------------------------------

.. index::
    single: Platform.sh;Debugging

By default, all Platform.sh environments use the same settings as the ``master``/``prod`` environment (aka the Symfony ``prod`` environment). This allows you to test the application in real-life conditions. It gives you the feeling of developing and testing directly on production servers, but without the risks associated with it. This reminds me of the good old days when we were deploying via FTP.

.. index::
    single: Symfony CLI;cloud:env:debug

In case of a problem, you might want to switch to the ``dev`` Symfony environment:

.. code-block:: terminal

    $ symfony cloud:env:debug

When done, move back to production settings:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    **Never** enable the ``dev`` environment and never enable the Symfony Profiler on the ``master`` branch; it would make your application really slow and open a lot of serious security vulnerabilities.

Testing Production Deployments before Deploying
-----------------------------------------------

Having access to the upcoming version of the website with production data opens up a lot of opportunities: from visual regression testing to performance testing. `Blackfire`_ is the perfect tool for the job.

Refer to the step about :doc:`Performance <29-performance>` to learn more about how you can use Blackfire to test your code before deploying.

Merging to Production
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

When you are satisfied with the branch changes, merge the code and the infrastructure back to the Git master branch:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

And deploy:

.. code-block:: terminal

    $ symfony cloud:deploy

When deploying, only the code and infrastructure changes are pushed to Platform.sh; the data are not affected in any way.

Cleaning up
-----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Finally, clean up by removing the Git branch and the Platform.sh environment:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Going Further

    * `Git branching`_;

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Git branching`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
