Project Teapot
==============

.. _teapot-introduction:

Introduction
------------

Project Teapot is a design proposal for a bare-metal cloud to run Kubernetes
on.

When OpenStack was first designed, 10 years ago, the cloud computing landscape
was a very different place. In the intervening period, OpenStack has amassed an
enormous installed base of many thousands of users who all depend on it
remaining essentially the same service, with backward-compatible APIs. If we
designed an open source cloud platform without those restrictions and looking
ahead to the 2020s, knowing everything we know today, what might it look like?
And how could we build it without starting from scratch, but using existing
open source technologies where possible? Project Teapot is one answer to these
questions.

Project Teapot is designed to run natively on Kubernetes, and to integrate with
Kubernetes clusters deployed by tenants. It provides only bare-metal compute
capacity, so that tenants can orchestrate all aspects of an application -- from
legacy VMs to cloud-native containers to workloads requiring custom hardware,
and everything in between -- through a single API that they can control.

It seems inevitable that numerous organisations are going to end up
implementing various subsets of this functionality just to deal with bare-metal
clusters in their own environment. By developing Teapot in the open, we would
give them a chance to reduce costs by collaborating on a common solution.

.. _teapot-goals:

Goals
-----

OpenStack's `mission
<https://governance.openstack.org/tc/resolutions/20160217-mission-amendment.html>`_
is to be ubiquitous; Teapot's is narrower. In the 2020s, Kubernetes will be
ubiquitous. However, Kubernetes' separation of responsibilities with the
underlying cloud mean that some important capabilities are considered out of
scope for it -- most obviously multi-tenancy of the sort provided by clouds,
allowing isolation from potentially malicious users (including innocuous users
who have had their workloads hacked by malicious third parties). Teapot's
primary mission is to fill those gaps with an open source solution, by
providing a cloud layer to manage a physical data center beneath Kubernetes.

In addition to mediating access to a physical data center, another important
role of clouds is to offer managed services (for example, a database as a
service). Teapot itself can be used to provide a managed service -- Kubernetes
(though it could equally be configured to provide fully user-controlled tenant
clusters). A secondary goal is to make Teapot a platform that cloud providers
could use to offer other kinds of managed service as well. Teapot is an easier
base than OpenStack on which to deploy such services because it is itself based
on Kubernetes.

.. _teapot-non-goals:

Non-Goals
---------

Teapot's design makes it suitable for deployments that require multi-tenancy
and are medium-sized or larger. Specifically, Teapot makes sense when tenants
are large enough to be able to utilise at least one (and usually more than one)
entire bare-metal server, because managing virtual machines is not a goal.

Smaller deployments that nevertheless require hard multi-tenancy (that is to
say, zero trust required between tenants) would be better off with OpenStack.

Smaller deployments that do not require hard multi-tenancy would be better off
running a single standalone Kubernetes cluster.

.. _teapot-design:

Design
------

The `Vision for OpenStack Clouds`_ states that the `physical data center
management function
<https://governance.openstack.org/tc/reference/technical-vision.html#basic-physical-data-center-management>`_
of a cloud must "[provide] the abstractions needed to deal with external
systems like :doc:`compute <compute>`, :doc:`storage <storage>`, and
:doc:`networking <networking>` hardware [including :doc:`load balancers
<load-balancing>` and :doc:`hardware security modules <key-management>`], the
:doc:`Domain Name System <dns>`, and :doc:`identity management systems <idm>`."
This proposal discusses implementation options for each of those classes of
systems.

Teapot also fulfils the `self-service
<https://governance.openstack.org/tc/reference/technical-vision.html#self-service>`_
requirements of a cloud, by providing multi-tenancy and :ref:`capacity
management <teapot-compute-reservation>`. In the Kubernetes model,
multi-tenancy is something that must be provided by the cloud layer.

Because Teapot targets Kubernetes as its tenant workload, it is able to
`provide applications control
<https://governance.openstack.org/tc/reference/technical-vision.html#application-control>`_
over the cloud using the standard Kubernetes interfaces (such as Ingress
resources and the Cluster Autoscaler). This greatly simplifies porting of many
workloads to and from other clouds.

Teapot is designed to be radically simpler than OpenStack to :doc:`install
<installation>` and operate. By running on the same technology stack as the
tenant clusters it deploys, it allows a common set of skills to be applied to
the operation of both applications and the underlying infrastructure. By
eschewing direct management of virtualisation it avoids having to shoehorn
bare-metal management into a virtualisation context or vice-versa, and
eliminates entire layers of networking abstractions.

At the same time, Teapot should be able to :doc:`interoperate with OpenStack
<openstack-integration>` when required so that each enhances the value of the
other without adding unnecessary layers of complexity.

Index
-----

.. toctree::
    compute
    storage
    networking
    load-balancing
    dns
    idm
    key-management
    installation
    openstack-integration

.. _Vision for OpenStack Clouds: https://governance.openstack.org/tc/reference/technical-vision.html
