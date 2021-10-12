Branching the Code
==================

There are many ways to organize the workflow of code changes in a project. But working directly on the Git master branch and deploying directly to production without testing is probably not the best one.

Testing is not just about unit or functional tests, it is also about checking the application behavior with production data. If you or your `stakeholders`_ can browse the application exactly as it will be deployed to end users, this becomes a huge advantage and allows you to deploy with confidence. It is especially powerful when non-technical people can validate new features.

We will continue doing all the work in the Git master branch in the next steps for simplicity sake and to avoid repeating ourselves, but let's see how this could work better.

Adopting a Git Workflow
-----------------------

One possible workflow is to create one branch per new feature or bug fix. It is simple and efficient.

Describing your Infrastructure
------------------------------

You might not have realized it yet, but having the infrastructure stored in files alongside of the code helps a lot. Docker and SymfonyCloud use configuration files to describe the project infrastructure. When a new feature needs an additional service, the code changes and the infrastructure changes are part of the same patch.

Creating Branches
-----------------

.. index::
    single: Git;branch
    single: Git;checkout

The workflow starts with the creation of a Git branch:

.. code-block:: bash
    :class: hide

    $ git branch -D sessions-in-redis || true

.. code-block:: bash

    $ git checkout -b sessions-in-redis

This command creates a ``sessions-in-redis`` branch from the ``master`` branch. It "forks" the code and the infrastructure configuration.

Storing Sessions in Redis
-------------------------

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: SymfonyCloud;Redis

As you might have guessed from the branch name, we want to switch session storage from the filesystem to a Redis store.

The needed steps to make it a reality are typical:

#. Create a Git branch;

#. Update the Symfony configuration if needed;

#. Write and/or update some code if needed;

#. Update the PHP configuration (add the Redis PHP extension);

#. Update the infrastructure on Docker and SymfonyCloud (add the Redis service);

#. Test locally;

#. Test remotely;

#. Merge the branch to master;

#. Deploy to production;

#. Delete the branch.

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

I'll let you test locally by browsing the website. As there are no visual changes and because we are not using sessions yet, everything should still work as before.

Deploying a Branch
------------------

.. index::
    single: SymfonyCloud;Environment

Before deploying to production, we should test the branch on the same infrastructure as the production one. We should also validate that everything works fine for the Symfony ``prod`` environment (the local website used the Symfony ``dev`` environment).

.. index::
    single: Symfony CLI;env:delete
    single: Symfony CLI;env:create

First, make sure to commit your changes to the new branch:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -m'Configure redis sessions'

Now, let's create a *SymfonyCloud environment* based on the *Git branch*:

.. code-block:: bash
    :class: hide

    $ symfony env:delete sessions-in-redis --no-interaction

.. code-block:: bash

    $ symfony env:create

This command creates a new environment as follows:

* The branch inherits the code and infrastructure from the current Git branch (``sessions-in-redis``);

* The data come from the master (aka production) environment by taking a consistent snapshot of all service data, including files (user uploaded files for instance) and databases;

* A new dedicated cluster is created to deploy the code, the data, and the infrastructure.

As the deployment follows the same steps as deploying to production, database migrations will also be executed. This is a great way to validate that the migrations work with production data.

The non-``master`` environments are very similar to the ``master`` one except for some small differences: for instance, emails are not sent by default.

.. index::
    single: Symfony CLI;open:remote

Once the deployment is finished, open the new branch in a browser:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote

Note that all SymfonyCloud commands work on the current Git branch. This command opens the deployed URL for the ``sessions-in-redis`` branch; the URL will look like ``https://sessions-in-redis-xxx.eu.s5y.io/``.

Test the website on this new environment, you should see all the data that you created in the master environment.

If you add more conferences on the ``master`` environment, they won't show up in the ``sessions-in-redis`` environment and vice-versa. The environments are independent and isolated.

If the code evolves on master, you can always rebase the Git branch and deploy the updated version, resolving the conflicts for both the code and the infrastructure.

.. index::
    single: Symfony CLI;env:sync

You can even synchronize the data from master back to the ``sessions-in-redis`` environment:

.. code-block:: bash
    :class: answers(y)

    $ symfony env:sync

Debugging Production Deployments before Deploying
-------------------------------------------------

.. index::
    single: SymfonyCloud;Debugging

By default, all SymfonyCloud environments use the same settings as the ``master``/``prod`` environment (aka the Symfony ``prod`` environment). This allows you to test the application in real-life conditions. It gives you the feeling of developing and testing directly on production servers, but without the risks associated with it. This reminds me of the good old days when we were deploying via FTP.

.. index::
    single: Symfony CLI;env:debug

In case of a problem, you might want to switch to the ``dev`` Symfony environment:

.. code-block:: bash

    $ symfony env:debug

When done, move back to production settings:

.. code-block:: bash

    $ symfony env:debug --off

.. warning::

    **Never** enable the ``dev`` environment and never enable the Symfony Profiler on the ``master`` branch; it would make your application really slow and open a lot of serious security vulnerabilities.

Testing Production Deployments before Deploying
-----------------------------------------------

Having access to the upcoming version of the website with production data opens up a lot of opportunities: from visual regression testing to performance testing. `Blackfire <https://blackfire.io>`_ is the perfect tool for the job.

Refer to the step about "Performance" to learn more about how you can use Blackfire to test your code before deploying.

Merging to Production
---------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

When you are satisfied with the branch changes, merge the code and the infrastructure back to the Git master branch:

.. code-block:: bash

    $ git checkout master
    $ git merge sessions-in-redis

And deploy:

.. code-block:: bash

    $ symfony deploy

When deploying, only the code and infrastructure changes are pushed to SymfonyCloud; the data are not affected in any way.

Cleaning up
-----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Finally, clean up by removing the Git branch and the SymfonyCloud environment:

.. code-block:: bash

    $ git branch -d sessions-in-redis
    $ symfony env:delete --env=sessions-in-redis --no-interaction

.. sidebar:: Going Further

    * `Git branching <https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell>`_;

    * `Redis docs <https://redis.io/documentation>`_.

.. _`stakeholders`: https://en.wikipedia.org/wiki/Project_stakeholder
