Teapot Networking
=================

In Project Teapot, tenant clusters are deployed exclusively on bare-metal
servers, which are under the complete control of the tenant. Therefore the
network itself must be the guarantor of multi-tenancy, with only untrusted
components running on tenant machines. (Trusted components can still run within
the management cluster.)

.. _teapot-networking-multi-tenancy:

Multi-tenant Network Model
--------------------------

Support for VLANs and VxLAN is ubiquitous in modern data center network
hardware, so this will be the basis for Teapot's networking. Each tenant will
be assigned one or more V(x)LANs. (Separate failure domains will likely also
have separate broadcast domains.) As machines are assigned to the tenant, the
Teapot controller will connect each to a private virtual network also assigned
to the tenant.

Small deployments can just use VLANs. Larger deployments may need VxLAN, and in
this case :abbr:`VTEP (VxLAN Tunnel EndPoint)`-capable edge switches and a
VTEP-capable router will be required.

This design frees the tenant clusters from being forced to use a particular
:abbr:`CNI (Container Network Interface)` plugin. Tenants are free to select a
networking overlay (e.g. Flannel, Cilium, OVN, &c.) or other CNI plugin (e.g.
Calico, Romana) of their choice within the tenant cluster, provided that it
does not need to be trusted by the network. (This would preclude solutions that
rely on advertising BGP/OSPF routes, although it's conceivable that one day
these advertisements could be filtered through a trusted component in the
management cluster and rebroadcast to the unencapsulated network -- this would
also be useful for :ref:`load balancing
<teapot-load-balancing-metallb-l3-tenant>` of Services.) If the tenant's CNI
plugin does create an overlay network, that technically means that packets will
be double-encapsulated, which is a Bad Thing when it occurs in VM-based
clusters, for several reasons:

* There is a performance overhead to encapsulating the packets on the
  hypervisor, and it also limits the ability to apply some performance
  optimisations (such as using SR-IOV to provide direct access to the NICs from
  the VMs by virtualising the PCIe bus).
* The extra overhead in each packet can cause fragmentation, and reduces the
  bandwidth available at the edge.
* Broadcast, multicast and unknown unicast traffic is flooded to all possible
  endpoints in the overlay network; doing this at multiple layers can increase
  network load.

However, these problems are significantly mitigated in the Teapot model:

* The performance cost of performing the second encapsulation is eliminated by
  offloading it to the network hardware.
* Encapsulation headers are carried only within the core of the network, where
  bandwidth is less scarce and frame sizes can be adjusted to prevent
  fragmentation.
* CNI plugins don't generally make significant use of broadcast or multicast.

.. _teapot-networking-provisioning:

Provisioning Network
--------------------

Generally bare-metal machines will need at least one interface connected to a
provisioning network in order to boot using :abbr:`PXE (Pre-boot execution
environment)`. Typically the provisioning network is required to be an untagged
VLAN.

PXE can be avoided by provisioning using virtual media (where the BMC attaches
a virtual disk containing the boot image to the host's USB), but hardware
support for doing this from Ironic is uneven (though rapidly improving) and it
is considerably slower than PXE. In addition, the Ironic agent typically
communicates over this network for purposes such as introspection of hosts or
cleaning of disks.

For the purpose of PXE booting, hosts could be left permanently connected to
the provisioning network provided they are isolated from each other (e.g. using
private VLANs). This would have the downside that the main network interface of
the tenant worker would have to appear on a tagged VLAN. However, the Ironic
agent's access to the Ironic APIs is unauthenticated, and therefore not safe to
be carried over networks that have hosts allocated to tenants connected to
them. This could occur over a separate network, but in any event hosts'
membership of this network will have to be changed dynamically in concert with
the baremetal provisioner.

The :abbr:`BMC (Baseboard management controller)`\ s will be connected to a
separate network that is reachable only from the management cluster.

.. _teapot-networking-storage:

Storage Network
---------------

When (optionally) used in combination with multi-tenant storage, machines will
need to also be connected to a separate storage network. The networking
requirements for this network are much simpler, as it does not need to be
dynamically managed. Each edge port should be isolated from all of the others
(using e.g. Private VLANs), regardless of whether they are part of the same
tenant. :abbr:`QoS (Quality of Service)` rules should ensure that no individual
machine can effectively deny access to others. Configuring the switches for the
storage network can be considered out of scope for Project Teapot, at least
initially, as the configuration need not be dynamic, but might be in scope for
the :doc:`installer <installation>`.

.. _teapot-networking-external:

External connections
--------------------

Workloads running in a tenant cluster can request to be exposed for incoming
external connections in a number of different ways. The Teapot cloud is
responsible for ensuring that each of these is possible.

The ``NodePort`` service type simply requires that the IP addresses of the
cluster members be routable from external networks.

For IPv4 support in particular, Teapot will need to be able to allocate public
IP addresses and route traffic for them to the appropriate networks.
Traditionally this is done using :abbr:`NAT (Network Address Translation)`
(e.g. Floating IPs in OpenStack). Users can specify an externalAddress to make
use of public IPs within their cluster, although there's no built-in way to
discover what IPs are available. Teapot should also have a way of exporting the
:doc:`reverse DNS records <dns>` for public IP addresses.

The ``LoadBalancer`` Service type uses an external :doc:`load balancer
<load-balancing>` as a front end. Traffic from the load balancer is directed
to a ``NodePort`` service within the tenant cluster.

Most managed Kubernetes services provide an Ingress controller that can set up
load balancing (including :abbr:`TLS (Transport Layer Security)` termination)
in the underlying cloud for HTTP(S) traffic, including automatically
configuring public IPs. If Teapot provided :ref:`such an Ingress controller
<teapot-load-balancing-ingress-controller>`, it might be a viable option to not
support public IPs at all for the ``NodePort`` service type. In this case, the
implementation of public IPs could be confined to the :ref:`load balancing API
<teapot-load-balancing-ingress-api>`, and the only stable public IP addresses
would be the Virtual IPs of the load balancers. Tenant IPv6 addresses could
easily be made publicly routable to provide direct access to ``NodePort``
services over IPv6 only, although this also comes with the caveat that some
clients may be tempted to rely on the IP of a Service being static, when in
fact the only safe way to reference it is via a :doc:`DNS name <dns>` exported
by ExternalDNS.

Implementation Options
----------------------

.. _teapot-networking-ansible:

Ansible Networking
~~~~~~~~~~~~~~~~~~

A good long-term implementation strategy might be to use ansible-networking to
directly configure the top-of-rack switches. This would be driven by a
Kubernetes controller running in the management cluster operating on a set of
Custom Resource Definitions (CRDs). The ansible-networking project supports a
wide variety of hardware already. A minimal proof of concept for this
controller `exists <https://github.com/bcrochet/physical-switch-operator>`_.

In addition to configuring the edge switches, a solution for public IPs and
other ways of exposing services is also needed. Future requirements likely
include configuring limited cross-tenant network connectivity, and access to
hardware load balancers and other data center hardware.

.. _teapot-networking-neutron:

OpenStack Neutron
~~~~~~~~~~~~~~~~~

A good short-term option might be to use a cut-down Neutron installation as an
implementation detail to manage the network. Using only the baremetal port
types in Neutron circumvents a lot of the complexity. Most of the Neutron
agents would not be required, so message queue--based RPC could be eliminated
or replaced with json-rpc (as it has been in Ironic for Metal³). Since only a
trusted service would be controlling network changes, Keystone authentication
would not be required either.

To ensure that Neutron itself could eventually be switched out, it would be
strictly confined behind a Kubernetes-native API, in much the same way as
Ironic is behind Metal³. The existing direct integration between Ironic and
Neutron would not be used, and nor could we rely on Neutron to provide an
integration point for e.g. :ref:`Octavia <teapot-load-balancing-octavia>` to
provide an abstraction over hardware load balancers.

The abstraction point would be the Kubernetes CRDs -- different controllers
could be chosen to manage custom resources (and those might in turn make use of
additional non-public CRDs), but we would not attempt to build controllers with
multiple plugin points that could lead to ballooning complexity.
