.. _production:

Handling Production Load
------------------------

In order to handle production load, you'll want to replace the sandbox's object store and PostgreSQL database with production grade storage systems. To do this, you'll need to modify your Flyte configuration to remove the sandbox datastores and reference new ones.

Flyte Configuration
*******************

A Flyte deployment contains around 50 kubernetes resources.
The Flyte team has chosen to use the "kustomize" tool to manage these configs.
Take a moment to read the `kustomize docs <https://github.com/kubernetes-sigs/kustomize>`_. Understanding kustomize will be important to modifying Flyte configurations.

The ``/kustomize`` directory in the `flyte repository <https://github.com/lyft/flyte/tree/master/kustomize>`_ is designed for use with ``kustomize`` to tailor Flyte deployments to your needs.
Important subdirectories are described below.

base
  The `base directory <https://github.com/lyft/flyte/tree/master/kustomize/base>`_ contains minimal configurations for each Flyte component. 

dependencies
  The `dependencies directory <https://github.com/lyft/flyte/tree/master/kustomize/dependencies>`_ contains deploy configurations for components like ``PostgreSQL`` that Flyte depends on.

These directories were designed so that you can modify them using ``kustomize`` to generate a custom Flyte deployment.
In fact, this is how we create the ``sandbox`` deployment.

Understanding the sandbox deployment will help you to create your own custom deployments.

Understanding the Sandbox
*************************

The sandbox deployment is managed by a set of kustomize `overlays <https://github.com/kubernetes-sigs/kustomize/blob/master/docs/glossary.md#overlay>`_ that alter the ``base`` configurations to compose the sandbox deployment. 

The sandbox overlays live in the `kustomize/overlays/sandbox <https://github.com/lyft/flyte/tree/master/kustomize/overlays/sandbox>`_ directory. There are overlays for each component, and a "flyte" overlay that aggregates the components into a single deploy file. 

**Component Overlays**
  For each modified component, there is an kustomize overlay at ``kustomize/overlays/sandbox/{{ component }}``.
  The overlay will typically reference the ``base`` for that component, and modify it to the needs of the sandbox.

  Using kustomize "patches", we add or override specific configs from the ``base`` resources. For example, in the "console" overlay, we specify a patch in the `kustomization.yaml <https://github.com/lyft/flyte/blob/master/kustomize/overlays/sandbox/console/kustomization.yaml>`_. This patch adds memory and cpu limits to the console deployment config.

  Each Flyte component requires at least one configuration file. The configuration files for each component live in the component overlay. For example, the FlyteAdmin config lives at `kustomize/overlays/sandbox/admindeployment/flyteadmin_config.yaml <https://github.com/lyft/flyte/blob/master/kustomize/overlays/sandbox/admindeployment/flyteadmin_config.yaml>`_. These files get included as Kubernetes configmaps and mounted into pods.

**Flyte Overlay**
  The ``flyte`` overlay is meant to aggregate the components into a single deployment file.
  The `kustomization.yaml <https://github.com/lyft/flyte/blob/master/kustomize/overlays/sandbox/flyte/kustomization.yaml>`_ in that directory lists the components to be included in the deploy.

  We run ``kustomize build`` against the ``flyte`` directory to generate the complete `sandbox deployment yaml <https://github.com/lyft/flyte/blob/master/deployment/sandbox/flyte_generated.yaml>`_ we used earlier to deploy Flyte sandbox.

Creating Your Own Deployment
****************************

Before you create a custom deployment, you'll need to `install kustomize <https://github.com/kubernetes-sigs/kustomize#kustomize>`_.

The simplest way to create your own custom deployment is to clone the sandbox deploy and modify it to your liking.

To do this, check out the ``flyte`` repo ::

  git clone https://github.com/lyft/flyte.git

Copy the sandbox configuration to a new directory on your machine, and enter the new directory ::

  cp -r flyte/kustomize/overlays/sandbox my-flyte-deployment
  cd my-flyte-deployment

Since the ``base`` files are not in your local copy, you'll need to make some slight modifications to reference the ``base`` files from our GitHub repository. :: 

  find . -name kustomization.yaml -print0 | xargs -0 sed -i.bak 's~../../../base~github.com/lyft/flyte/kustomize/base~'
  find . -name kustomization.yaml -print0 | xargs -0 sed -i.bak 's~../../../dependencies~github.com/lyft/flyte/kustomize/dependencies~'
  find . -name '*.bak' | xargs rm

You should now be able to run kustomize against the ``flyte`` directory ::

  kustomize build flyte > flyte_generated.yaml

This will generate a deployment file identical to the sandbox deploy, and place it in a file called ``flyte_generated.yaml`` 

Going Beyond the Sandbox
************************

Let's modify the sandbox deployment to use cloud providers for the database and object store. 

Production Grade Database
*************************

The ``FlyteAdmin`` and ``DataCatalog`` components rely on PostgreSQL to store persistent records. 

In this section, we'll modify the Flyte deploy to use a remote PostgreSQL database instead.

First, you'll need to set up a reliable PostgreSQL database. The easiest way achieve this is to use a cloud provider like AWS `RDS <https://aws.amazon.com/rds/postgresql/>`_, GCP `Cloud SQL <https://cloud.google.com/sql/docs/postgres/>`_, or Azure `PostgreSQL <https://azure.microsoft.com/en-us/services/postgresql/>`_ to manage the PostgreSQL database for you. Create one and make note of the username, password, endpoint, and port. 

Next, remove old sandbox database by opening up the ``flyte/kustomization.yaml`` file and deleting database component. ::

  - github.com/lyft/flyte/kustomize/dependencies/database

With this line removed, you can re-run ``kustomize build flyte > flyte_generated.yaml`` and see that the the postgres deployment has been removed from the ``flyte_generated.yaml`` file.

Now, let's re-configure ``FlyteAdmin`` to use the new database.
Edit the ``admindeployment/flyteadmin_config.yaml`` file, and change the ``storage`` key like so ::

    database:
      host: {{ your-database.endpoint }}
      port: {{ your database port }}
      username: {{ your_database_username }}
      password: {{ your_database_password }}
      dbname: flyteadmin

Do the same thing in ``datacatalog/datacatalog_config.yaml``, but use the dbname ``datacatalog`` ::

    database:
      host: {{ your-database.endpoint }}
      port: {{ your database port }}
      username: {{ your_database_username }}
      password: {{ your_database_password }}
      dbname: datacatalog

Note: *You can mount the database password into the pod and use the "passwordPath" config to point to a file on disk instead of specifying the password here*

Next, remove the "check-db-ready" init container from `admindeployment/admindeployment.yaml <https://github.com/lyft/flyte/blob/master/kustomize/overlays/sandbox/admindeployment/admindeployment.yaml#L10-L14>`_. This check is no longer needed.

Production Grade Object Store
*****************************

``FlyteAdmin``, ``FlytePropeller``, and ``DataCatalog`` components rely on an Object Store to hold files.

In this section, we'll modify the Flyte deploy to use `AWS S3 <https://aws.amazon.com/s3/>`_ for object storage.
The process for other cloud providers like `GCP GCS <https://cloud.google.com/storage/>`_ should be similar.

To start, `create an s3 bucket <https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html>`_.

Next, remove the old sandbox object store by opening up the ``flyte/kustomization.yaml`` file and deleting the storage line. ::

  - github.com/lyft/flyte/kustomize/dependencies/storage

With this line gone, you can re-run ``kustomize build flyte > flyte_generated.yaml`` and see that the sandbox object store has been removed from the ``flyte_generated.yaml`` file.

Next, open the configs ``admindeployment/flyteadmin_config.yaml``, ``propeller/config.yaml``, ``datacatalog/datacatalog_config.yaml`` and look for the ``storage`` configuration.

Change the ``storage`` configuration in each of these configs to use your new s3 bucket like so ::

    storage:
      type: s3
      container: {{ YOUR-S3-BUCKET }}
      connection:
        auth-type: accesskey
        access-key: {{ YOUR_AWS_ACCESS_KEY }}
        secret-key: {{ YOUR_AWS_SECRET_KEY }}
        region: {{ YOUR-AWS-REGION }}

Note: *To use IAM roles for authentication, switch to the "iam" auth-type.*

Next, open ``propeller/plugins/config.yaml`` and remove the `default-env-vars <https://github.com/lyft/flyte/blob/master/kustomize/overlays/sandbox/propeller/plugins/config.yaml#L13-L15>`_ (no need to replace them, the default behavior is sufficient).

Now if you re-run ``kustomize build flyte > flyte_generated.yaml``, you should see that the configmaps have been updated.

Run ``kubectl apply -f flyte_generated.yaml`` to deploy these changes to your cluster for a production-ready deployment.

Dynamically Configured Projects
*******************************

As your Flyte user-base evolves, adding new projects is as simple as registering them through the cli ::

    flyte-cli -h {{ your-flyte-admin-host.com }} register-project --identifier myuniqueworkflow --name FriendlyWorkflowName 

A cron which runs at the cadence specified in flyteadmin config will ensure that all the kubernetes resources necessary for the new project are created and new workflows can successfully
be registered and executed under the new project.

