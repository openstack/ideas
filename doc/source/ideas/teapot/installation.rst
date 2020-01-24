Teapot Installation
===================

In a sense, the core of Teapot is simply an application running in a Kubernetes
cluster (the management cluster). This is a great advantage for ease of
installation, because Kubernetes is renowned for its simplicity in
bootstrapping. Many, many (perhaps too many) tools already exist for
bootstrapping a Kubernetes cluster, so there is no need to reinvent them.

However, Teapot is designed to be the system that provides cloud services to
bare-metal Kubernetes clusters, and while it is possible to run the management
cluster on another cloud (such as OpenStack), it is likely in most instances to
be self-hosted on bare metal. This presents a unique bootstrapping challenge.

OpenStack does not define an 'official' installer, largely due to the plethora
of configuration management tools that different users preferred. Teapot does
not have the same issue, as it standardises on Kubernetes as the *lingua
franca*. There should be a single official installer and third parties are
encouraged to add extensions and customisations by adding Resources and
Operators through the Kubernetes API.

Implementation Options
----------------------

Metal³
~~~~~~

`Metal³`_ is designed to bootstrap standalone bare-metal clusters, so it can be
used to install the management cluster. There are multiple ways to do this. One
is to use the `Cluster API`_ on a bootstrap VM, and then pivot the relevant
resources into the cluster. The OpenShift installer takes a slightly different
approach, again using a bootstrap VM, but creating the master nodes initially
using Terraform and then creating BareMetalHost resources marked as 'externally
provisioned' for them in the cluster.

One inevitable challenge is that the initial bootstrap VM must be able to
connect to the :ref:`management and provisioning networks
<teapot-networking-provisioning>` in order to begin the installation. That
makes it difficult to simply run from a laptop, which makes installing a small
proof-of-concept cluster harder than anyone would like. (This is inherent to
the bare-metal environment and also a problem for OpenStack installers). If a
physical host must be used as the bootstrap, reincorporating that hardware into
the actual cluster once it is up and running should at least be simpler on
Kubernetes.

Airship
~~~~~~~

Airship_ 2.0 uses Metal³ and the Cluster API to provision Kubernetes clusters
on bare metal. It also provides a declarative way of repeatably setting the
initial configuration and workloads of the deployed cluster, along with a rich
document layering and substitution model (based on Kustomize). This might be
the simplest existing way of defining what a Teapot installation looks like
while allowing distributors and third-party vendors a clear method for
providing customisations and add-ons.

Teapot Operator
~~~~~~~~~~~~~~~

A Kubernetes operator for managing the deployment and configuration of the
Teapot components could greatly simplify the installation process. This is not
incompatible with using Airship (or indeed any other method) to define the
configuration, as Helm would just create the top-level custom resource(s)
controlled by the operator, instead of lower-level resources for the individual
components.

.. _Metal³: https://metal3.io/
.. _Cluster API: https://github.com/kubernetes-sigs/cluster-api#readme
.. _Airship: https://www.airshipit.org/
