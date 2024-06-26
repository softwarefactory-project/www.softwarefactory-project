Monocle Operator - OLM
######################

:date: 2023-05-15
:category: blog
:authors: Fabien Boucher

.. raw:: html

   <style type="text/css">

     .literal {
       border-radius: 6px;
       padding: 1px 1px;
       background-color: rgba(27,31,35,.05);
     }

   </style>

This post aims to introduce the `Operator Lifecycle Management (OLM)`_
and how we integrated the `Monocle Operator`_ as an OLM package into the
`operatorhub.io`_ catalog.

This article is a follow up post of "`Monocle Operator - Phase 1 - Basic
Install`_".

What is OLM
===========

The Operator Lifecycle Management (OLM) is an approach to simplify
Kubernetes operators deployment, updates and lifecycle management.

OLM is composed of two main components:

-  **Catalogs**: These are collections of Operators that can be
   installed on a Kubernetes cluster. Catalogs can be public or private,
   and can be hosted on container registries. Each Operator in a catalog
   has a corresponding manifest that describes its deployment,
   configuration, and management. Catalogs allow users to easily
   discover, install and upgrade Operators on their cluster.

-  **The Operator Lifecycle Manager**: This is the control plane
   component of OLM that manages the installation, upgrade, and removal
   of Operators on a Kubernetes cluster. The Operator Lifecycle Manager
   is responsible for ensuring that Operators are deployed and managed
   according to their defined lifecycle. It monitors the status of
   Operators, handles upgrades and rollbacks, and ensures that
   dependencies between Operators are resolved correctly.

OLM can be seen as a Linux Package Manager like **DNF**, indeed:

-  both rely on a manifest to ensure proper installation and
   configuration of the package (the operator).
-  OLM and DNF package managers ensure that dependencies are resolved
   and the component is deployed and managed according to its defined
   lifecycle.
-  both systems offer a standardized approach to managing software
   components, improving system stability and efficiency.

Here is a `glossary`_ of the OLM terminology.

OLM installation
================

The `operator-sdk`_ tool provides a command to deploy OLM on a
Kubernetes deployment. This command creates various k8s resources to
spawn OLM components and associated roles, role bindinds, service users,
...

.. code-block:: shell

   operator-sdk olm install

Note that the Ansible role `ansible-microshift-role`_ provides an easy
way to deploy a lightweight OpenShift environment (using `Microshift`_)
with OLM enabled.

This command creates two namespaces:

-  **olm**: It contains the OLM system with the **catalog-operator**,
   **olm-operator** and the **packageserver** deployments.
-  **operators**: It is the placeholder where one can subscribe to one
   or more operators. No resource is populated here by the OLM
   installation.

.. code-block:: shell

   kubectl -n olm get deployment
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   catalog-operator   1/1     1            1           17d
   olm-operator       1/1     1            1           17d
   packageserver      2/2     2            2           17d

Now we are able to extend our k8s instance by installing operators.

OLM usage
=========

In order to learn more about OLM, we will deploy the `cert-manager
operator`_ from OLM.

The OLM installation should come with the **Community Operators**
catalog installed:

.. code-block:: shell

   kubectl -n olm get catalogsources operatorhubio-catalog
   NAME                    DISPLAY               TYPE   PUBLISHER        AGE
   operatorhubio-catalog   Community Operators   grpc   OperatorHub.io   17d

   kubectl -n olm get -o json catalogsources operatorhubio-catalog | jq '.spec'
   {
     "displayName": "Community Operators",
     "grpcPodConfig": {
       "securityContextConfig": "restricted"
     },
     "image": "quay.io/operatorhubio/catalog:latest",
     "publisher": "OperatorHub.io",
     "sourceType": "grpc",
     "updateStrategy": {
       "registryPoll": {
         "interval": "60m"
       }
     }
   }

Then we can explore the catalog for available operators:

.. code-block:: shell

   # There is more than 300 operators listed so let's grep for cert-manager
   kubectl -n olm get packagemanifests | grep cert-manager
   cert-manager                               Community Operators   17d

A **PackageManifest** resource describes the following:

-  The name and description of the package being managed.
-  The package's installation process and any dependencies required.
-  The default channel and available channels through which different
   versions of the package can be installed.
-  A list of all versions of the package available through each channel.
-  The latest version of the package available by channel
   (``currentCSV``).
-  A list of CRDs that are installed along with the package.
-  A list of global configuration variables for the package.

The **PackageManifest** resource could be heavy to inspect, here are
some commands to help:

.. code-block:: shell

   # Show the package provider
   kubectl -n olm get -o json packagemanifests cert-manager | jq '.status.provider'
   {
     "name": "The cert-manager maintainers",
     "url": "https://cert-manager.io/"
   }

   # Show available channels for that package
   kubectl -n olm get -o json packagemanifests cert-manager | jq '.status.channels[].name'
   "candidate"
   "stable"

   # Show the default install channel of that package
   kubectl -n olm get -o json packagemanifests cert-manager | jq '.status.defaultChannel'
   "stable"

   # Last version available (package head) in the stable channel
   kubectl -n olm get -o json packagemanifests cert-manager | jq '.status.channels[] | select(.name == "stable") | .currentCSV'
   "cert-manager.v1.11.0"

   # Versions from the stable channel
   kubectl -n olm get -o json packagemanifests cert-manager | jq '.status.channels[] | select(.name == "stable") | .entries'
   [
     {
       "name": "cert-manager.v1.11.0",
       "version": "1.11.0"
     },
     {
       "name": "cert-manager.v1.10.2",
       "version": "1.10.2"
     },
     ...
   ]

   # And finally, to show the CSV of the last stable version
   kubectl -n olm get -o json  packagemanifests cert-manager | jq '.status.channels[] | select(.name == "stable") | .currentCSVDesc'

The **PackageManifest** is built from a list of `ClusterServiceVersion
definition`_. The **ClusterServiceVersion** resource defines information
that is required to run the Operator, like the RBAC rules it requires
and which custom resources (CRs) it manages or depends on.

To install the **cert-manager** operator from the **stable** channel we
need to create a `Subscription`_. It describes which channel of an
operator package to subscribe to, and whether to perform updates
automatically or manually.

Create the file *cert-manager.yaml*:

.. code-block:: yaml

   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: my-cert-manager
     namespace: operators
   spec:
     channel: stable
     name: cert-manager
     source: operatorhubio-catalog
     sourceNamespace: olm
     # By default is automatic upgrade plan
     # installPlanApproval: Manual

Then apply it with:

.. code-block:: shell

   # Apply the subscription
   kubectl apply -f cert-manager.yaml

   # Get the subscription
   kubectl -n operators get sub
   NAME                  PACKAGE            SOURCE                  CHANNEL
   my-cert-manager       cert-manager       operatorhubio-catalog   stable

   # Ensure the CSV is now available
   kubectl -n operators get csv
   NAME                       DISPLAY            VERSION   REPLACES                   PHASE
   cert-manager.v1.11.0       cert-manager       1.11.0    cert-manager.v1.10.2       Succeeded

Note that an `InstallPlan`_ resource has been created too. This is where
you can inspect installation steps on the operator. This resource could
be inspected in case the requested operator failed to be installed, for
instance when the ``csv`` resource has not been created.

.. code-block:: shell

   kubectl -n operators describe installplan install-tkcrn

By default the **Subscription** set the **installPlanApproval** as
automatic. However if you decide to set it as manual, when OLM detects a
possible upgrade (because of a new version available in the ``stable``
channel), then the ``InstallPlan`` will need to be manually updated to
approve the upgrade. The process is described `here`_.

Beside the fact that the ``cert-manager.v1.11.0`` CSV phase is
``Succeeded`` we can verify that the ``cert-manager`` operator is
running:

.. code-block:: shell

   kubectl -n operators get all | grep cert-manager
   pod/cert-manager-68c79ccf94-hkbp8                               1/1     Running   0          62m
   pod/cert-manager-cainjector-86c79dd959-q6x2q                    1/1     Running   0          62m
   pod/cert-manager-webhook-b685d8cd4-9q6jj                        1/1     Running   0          62m
   service/cert-manager                                          ClusterIP   10.43.98.149    <none>        9402/TCP   63m
   service/cert-manager-webhook                                  ClusterIP   10.43.18.198    <none>        443/TCP    63m
   service/cert-manager-webhook-service                          ClusterIP   10.43.34.128    <none>        443/TCP    62m
   deployment.apps/cert-manager                               1/1     1            1           62m
   deployment.apps/cert-manager-cainjector                    1/1     1            1           62m
   deployment.apps/cert-manager-webhook                       1/1     1            1           62m
   replicaset.apps/cert-manager-68c79ccf94                               1         1         1       62m
   replicaset.apps/cert-manager-cainjector-86c79dd959                    1         1         1       62m
   replicaset.apps/cert-manager-webhook-b685d8cd4                        1         1         1       62m

The requested operator is installed in the same namespace as its
``Subscription``.

We can also ensure that the CRDs provided by the operator are available:

.. code-block:: shell

   kubectl api-resources | grep cert-manager
   challenges                                     acme.cert-manager.io/v1                      true         Challenge
   orders                                         acme.cert-manager.io/v1                      true         Order
   certificaterequests               cr,crs       cert-manager.io/v1                           true         CertificateRequest
   certificates                      cert,certs   cert-manager.io/v1                           true         Certificate
   clusterissuers                                 cert-manager.io/v1                           false        ClusterIssuer
   issuers                                        cert-manager.io/v1                           true         Issuer

Finally, let's create a ``namespace`` and request an ``Issuer`` resource
to the ``cert-manager operator``:

Create the file *issuer.yaml*:

.. code-block:: yaml

   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: example-issuer
   spec:
     selfSigned: {}

Then apply the resource in a new namespace:

.. code-block:: shell

   kubectl ceate ns test-cert-manager

   kubectl -n test-cert-manager apply -f issuer.yaml
   issuer.cert-manager.io/example-issuer created

   kubectl -n test-cert-manager get issuers
   NAME             READY   AGE
   example-issuer   True    7s

Packaging Monocle for OLM
=========================

Recently we wrote an `Operator`_ for the Monocle project and we were
curious about how to leverage OLM to make it easily consumable.

An `operator.yaml`_ file was generated by the
``kustomize build config/default`` command, then it was possible to
apply the Monocle CRD and to *install* the required resources
(namespace, serviceuser, roles, role bindings, deployments, ...) to get
the operator running.

From there the process was to create the `bundle`_ (or the package)
using the ``Makefile``'s ``bundle`` target:

.. code-block:: shell

   make bundle

This creates a directory called **bundle** which contains some
sub-directories:

-  *manifests*: containing mainly the CRD(s), and the
   ClusterServiceVersion.
-  *metadata*: this is some annotations to describe the bundle.
-  *tests/scorecard*: this describes various validation tests to be
   performed on the bundle.

Now we would like to **validate our bundle**, so we need to perform the
following steps.

First we need to **build and publish** the ``bundle``'s container image.
To do so, our ``Makefile`` provides the ``bundle-build`` and
``bundle-push`` targets:

.. code-block:: shell

   export BUNDLE_IMG=quay.io/change-metrics/monocle-operator-bundle:v0.0.1
   make bundle-build bundle-push

Then we can use the ``operator-sdk run bundle`` `command`_ to **validate
the bundle**. The command drives these steps:

-  Create an ``operator catalog`` containing only our ``bundle``
-  Run the ``registry`` pod to serve the new ``catalog``
-  Create a ``CatalogSource`` resource to make the new ``catalog``
   available
-  Create a ``Subscription`` and wait for the ``ClusterServiceVersion``
   to be available.

Note that this command needs to pull the bundle image from a real
container registry thus we run ``bundle-push`` to publish it. Running a
`local registry`_ could ease that process by avoiding the need to push
the bundle image on ``dockerhub`` or ``quay.io``.

.. code-block:: shell

   kubectl create ns test-bundle
   oc adm policy add-scc-to-user privileged system:serviceaccount:test-bundle:default
   export BUNDLE_IMG=quay.io/change-metrics/monocle-operator-bundle:v0.0.1
   operator-sdk run bundle $BUNDLE_IMG --namespace test-bundle --security-context-config restricted
   INFO[0010] Creating a File-Based Catalog of the bundle "quay.io/change-metrics/monocle-operator-bundle:v0.0.1"
   INFO[0011] Generated a valid File-Based Catalog
   INFO[0016] Created registry pod: quay-io-change-metrics-monocle-operator-bundle-v0-0-1
   INFO[0016] Created CatalogSource: monocle-operator-catalog
   INFO[0016] OperatorGroup "operator-sdk-og" created
   INFO[0016] Created Subscription: monocle-operator-v0-0-1-sub
   INFO[0022] Approved InstallPlan install-74dzl for the Subscription: monocle-operator-v0-0-1-sub
   INFO[0022] Waiting for ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" to reach 'Succeeded' phase
   INFO[0022]   Waiting for ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" to appear
   INFO[0035]   Found ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" phase: Pending
   INFO[0036]   Found ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" phase: InstallReady
   INFO[0037]   Found ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" phase: Installing
   INFO[0046]   Found ClusterServiceVersion "test-bundle/monocle-operator.v0.0.1" phase: Succeeded
   INFO[0047] OLM has successfully installed "monocle-operator.v0.0.1"

The ``test-bundle`` namespace can be cleaned using:

.. code-block:: shell

   operator-sdk cleanup --namespace test-bundle monocle-operator
   INFO[0001] subscription "monocle-operator-v0-0-1-sub" deleted
   INFO[0001] customresourcedefinition "monocles.monocle.monocle.change-metrics.io" deleted
   INFO[0002] clusterserviceversion "monocle-operator.v0.0.1" deleted
   INFO[0002] catalogsource "monocle-operator-catalog" deleted
   INFO[0003] operatorgroup "operator-sdk-og" deleted
   INFO[0003] Operator "monocle-operator" uninstalled

At that point, we have a *validated* ``bundle``. The next step is to
publish/distribute it. To do so, either:

-  we need to `maintain a catalog image`_.
-  or we distribute the bundle via an existing catalog like
   `operatorhub.io`_.

.. _monocle-operator-on-operatorhubio:

Monocle operator on OperatorHub.io
==================================

We decided to propose the operator to the **Community Catalog**. This
section explains the process we followed to publish the Monocle Operator
on `operatorhub.io`_.

First, we ensured that the **required bundle CSV fields are present**
(see the `required fields`_). If not the CSV template needs to be
adapted in
``config/manifests/bases/monocle-operator.clusterserviceversion.yaml``.

The ``make bundle`` command must be run to apply changes to the
``bundle`` directory.

We also **validated the bundle** with the ``validate`` command:

.. code-block:: shell

   operator-sdk bundle validate ./bundle --select-optional suite=operatorframework
   INFO[0000] All validation tests have completed successfully

Furthermore we **run the scorecard validation** (built-in basic and OLM
tests. See `scorecard`_):

.. code-block:: shell

   operator-sdk scorecard bundle -o text --pod-security restricted -n scorecard

Finally we created a `Pull Request`_ on the
`k8s-operatorhub/community-operators`_ repository.

This Pull Request includes a copy of the ``bundle`` located in a new
directory called ``operators/monocle-operator/0.0.1``. The
``operators/monocle-operator/ci.yaml`` file was also needed to define
`various settings`_ for the operatorhub.io's CI pipelines.

After some back and forth, mainly thanks to the operatorhub.io's CI
catching issues, the Monocle Operator Pull Request landed and few
minutes later (propably the time required by the CD pipeline to update
and publish the catalog) it `appeared on the operatorhub.io website`_,
and was available on our Microshift installation:

.. code-block:: shell

   kubectl -n olm get packagemanifests monocle-operator
   NAME               CATALOG               AGE
   monocle-operator   Community Operators   18d

Feel free to refer to the upstream `Add your operator - documentation`_
for more details.

To conclude
===========

As we are working closer with OpenShift and the Go Operator pattern, our
team decided to investigate OLM to gather knowledge. After some readings
and experimentations we were able to figure out how to leverage OLM to
distribute a Kubernetes operator. We used the Monocle Operator to
perform that experimentation because it was almost *ready to bundle*.
This experimentation will help us to better align our further
developments for **SF 4.X** aka the `sf-operator`_.

.. _Operator Lifecycle Management (OLM): https://olm.operatorframework.io/
.. _Monocle Operator: https://github.com/change-metrics/monocle-operator
.. _operatorhub.io: https://operatorhub.io
.. _Monocle Operator - Phase 1 - Basic Install: https://www.softwarefactory-project.io/monocle-operator-phase-1-basic-install.html
.. _glossary: https://olm.operatorframework.io/docs/glossary/
.. _operator-sdk: https://sdk.operatorframework.io/
.. _ansible-microshift-role: https://github.com/openstack-k8s-operators/ansible-microshift-role
.. _Microshift: https://github.com/openshift/microshift
.. _cert-manager operator: https://operatorhub.io/operator/cert-manager
.. _ClusterServiceVersion definition: https://docs.openshift.com/container-platform/4.12/operators/understanding/olm-common-terms.html#olm-common-terms-csv_olm-common-terms
.. _Subscription: https://olm.operatorframework.io/docs/concepts/crds/subscription/
.. _InstallPlan: https://olm.operatorframework.io/docs/concepts/crds/installplan/
.. _here: https://olm.operatorframework.io/docs/concepts/crds/subscription/#manually-approving-upgrades-via-subscriptions
.. _Operator: https://github.com/change-metrics/monocle-operator
.. _operator.yaml: https://github.com/change-metrics/monocle-operator/tree/6b8a02f9087f83798f732ede85cbe35c0304cb58/install
.. _bundle: https://olm.operatorframework.io/docs/glossary/#bundle
.. _command: https://sdk.operatorframework.io/docs/cli/operator-sdk_run_bundle/
.. _local registry: https://hub.docker.com/_/registry
.. _maintain a catalog image: https://sdk.operatorframework.io/docs/olm-integration/tutorial-bundle/#deploying-bundles-in-production
.. _required fields: https://k8s-operatorhub.github.io/community-operators/packaging-required-fields/
.. _scorecard: https://sdk.operatorframework.io/docs/testing-operators/scorecard/
.. _Pull Request: https://github.com/k8s-operatorhub/community-operators/pull/2668
.. _k8s-operatorhub/community-operators: https://github.com/k8s-operatorhub/community-operators
.. _various settings: https://k8s-operatorhub.github.io/community-operators/operator-ci-yaml/#operator-versioning
.. _appeared on the operatorhub.io website: https://operatorhub.io/operator/monocle-operator
.. _Add your operator - documentation: https://k8s-operatorhub.github.io/community-operators/
.. _sf-operator: https://github.com/softwarefactory-project/sf-operator
