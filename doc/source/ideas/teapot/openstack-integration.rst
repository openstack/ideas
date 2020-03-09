Teapot and OpenStack
====================

Many potential users of Teapot have large existing OpenStack deployments.
Teapot is not intended to be a wholesale replacement for OpenStack -- it does
not deal with virtualisation at all, in fact -- so it is important that the two
complement each other.

.. _teapot-openstack-managed-services:

Managed Services
----------------

A goal of Teapot is to make it easier for cloud providers to offer managed
services to tenants. Attempts to do this in OpenStack, such as Trove_, have
mostly foundered. The Kubernetes Operator pattern offers the most promising
ground for building such services in future, and since Teapot is
Kubernetes-native it would be well-placed to host them.

Building a thin OpenStack-style ReST API over such services would allow their
use from an OpenStack cloud (presumably sharing, or federated to, the same
Keystone) simultaneously. And, in fact, most such services could be decoupled
from Teapot altogether and run in a generic Kubernetes cluster so that they
could benefit users of either cloud type even absent the other.

Teapot's :ref:`load balancing API <teapot-load-balancing-ingress-api>` would
arguably already be a managed service. :ref:`Octavia
<teapot-load-balancing-octavia>` could possibly use it as a back-end as a first
example.

.. _teapot-openstack-side-by-side:

Side-by-side Clouds
-------------------

Teapot should be co-installable alongside an existing OpenStack cloud to
provide additional value. In this configuration, the Teapot cloud would use the
OpenStack cloud's :ref:`Keystone <teapot-idm-keystone>` and any services that
are expected to be found in the catalog (e.g. :ref:`Manila
<teapot-storage-manila>`, :ref:`Cinder <teapot-storage-cinder>`,
:ref:`Designate <teapot-dns-designate>`).

An OpenStack-style ReST API in front of Teapot would allow users of the
OpenStack cloud to create and manage bare-metal Kubernetes clusters in much the
same way they do today with Magnum.

Tenants would need a way to connect their Neutron networks in OpenStack to the
Kubernetes clusters. Since Teapot tenant networks are :ref:`just V(x)LANs
<teapot-networking-multi-tenancy>`, this could be accomplished by adding those
networks as provider networks in Neutron, and allowing the correct tenants to
connect to them via Neutron routers. This should be sufficient for the main use
case, which would be running parts of an application in a Kubernetes cluster
while other parts remain in OpenStack VMs.

However, the ideal for this type of deployment would be to allow servers to be
dynamically moved between the OpenStack and Teapot clouds. Sharing inventory
with OpenStack's Ironic might be simple enough -- if Metal³ was configured to
use the OpenStack cloud's Ironic then a small component could claim hosts in
OpenStack Placement and create corresponding BareMetalHost objects in Teapot.
Both clouds would end up manipulating the top-of-rack switch configuration for
a host, but presumably only at different times.

Switching hosts between acting as OpenStack compute nodes and being available
to Teapot tenants would be more complex, since it would require interaction
with the tool managing the OpenStack deployment, of which there are many.
However, supporting autoscaling between the two is probably unnecessary.
Manually moving hosts between the clouds should be manageable, since no changes
to the physical network cabling would be required. Separate :ref:`provisioning
networks <teapot-networking-provisioning>` would need to be maintained, since
the provisioner needs control over DHCP.

.. _teapot-openstack-on-teapot:

OpenStack on Teapot
-------------------

To date, the most popular OpenStack installers have converged on Ansible as a
deployment tool because the complexity of OpenStack needs tight control over
the workflow that purely declarative tools struggle to match. However,
Kubernetes Operators present a declarative alternative that is nonetheless
equally flexible. Even without Operators, Airship_ and StarlingX_ are both
installing OpenStack on top of Kubernetes. It seems likely that in the future
this will be a popular way of layering things, and Teapot is well-placed to
enable it since it provides bare-metal hosts running Kubernetes.

For a large, shared OpenStack cloud, this would likely be best achieved by
running the OpenStack control plane components inside the Teapot management
cluster. Sharing of services would then be similar to the side-by-side case.
OpenStack Compute nodes or e.g. Ceph storage nodes could be deployed using
`Metal³`_. This effectively means building an OpenStack installation/management
system similar to a TripleO undercloud but based on Kubernetes.

There is a second use case, for running small OpenStack installations (similar
to StarlingX) within a tenant. In these cases, the tenant OpenStack would still
need to access storage from the Teapot cloud. This could possibly be achieved
by federating the tenant Keystone to Teapot's Keystone and using hierarchical
multi-tenancy so that projects in the tenant Keystone are actually sub-projects
of the tenant's project in the Teapot Keystone. (The long-dead `Trio2o
<https://opendev.org/x/trio2o#trio2o>`_ project also offered a potential
solution in the form of an API proxy, but probably not one worth resurrecting.)
Use of an overlay network (e.g. OVN) would be required, since the tenant would
have no access to the underlying network hardware. Some integration between the
tenant's Neutron and Teapot would need to be built to allow ingress traffic.


.. _Trove: https://docs.openstack.org/trove/
.. _Airship: https://www.airshipit.org/
.. _StarlingX: https://www.starlingx.io/
.. _Metal³: https://metal3.io/
