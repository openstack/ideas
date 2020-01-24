Teapot Load Balancing
=====================

Load balancers are one of the things that Kubernetes expects to be provided by
the underlying cloud. No multi-tenant bare-metal solutions for this exist, so
project Teapot would need to provide one. Ideally an external load balancer
would act as an abstraction over what could be either a tenant-specific
software load balancer or multi-tenant-safe access to a hardware (or virtual)
load balancer.

There are two ways for an application to request an external load balancer in
Kubernetes. The first is to create a Service_ with type |LoadBalancer|_. This
is the older way of doing things but is still useful for lower-level plumbing,
and may be required for non-HTTP(S) protocols. The preferred (though nominally
beta) way is to create an Ingress_. The Ingress API allows for more
sophisticated control (such as adding abbr:`TLS (Transport Layer Security)`
termination), and can allow multiple services to share a single external load
balancer (including across different DNS names), and hence a single IP address.

Most managed Kubernetes services provide an Ingress controller that can set up
external load balancers, including TLS termination, using the underlying
cloud's services. Without this, tenants can still use an Ingress controller,
but it would have to be one that uses resources available to the tenant, such
as by running software load balancers in the tenant cluster.

When using a Service of type |LoadBalancer| (rather than an Ingress), there is
no standardised way of requesting TLS termination (some cloud providers permit
it using an annotation), so supporting this use case is not a high priority.
The |LoadBalancer| Service type in general should be supported, however (though
there are existing Kubernetes offerings where it is not).

Implementation options
----------------------

The choices below are not mutually exclusive. An administrator of a Teapot
cloud and their tenants could each potentially choose from among several
available options.

.. _teapot-load-balancing-metallb-l2:

MetalLB (Layer 2) on tenant cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The MetalLB_ project provides two ways of doing load balancing for bare-metal
clusters. One requires control over only layer 2, although it really only
provides the high-availability aspects of load balancing, not actual balancing.
All incoming traffic for each service is directed to a single node; from there
kubeproxy distributes it to the endpoints that handle it. However, should the
node die, traffic rapidly fails over to another node.

This form of load balancing does not support offloading TLS termination,
results in large amounts of East-West traffic, and consumes resources from the
guest cluster.

Tenants could decide to use this unilaterally (i.e. without the involvement of
the management cluster or its administrators). However, using MetalLB restricts
the choice of CNI plugins -- for example it does not work with OVN. A
pre-requisite to use it would be that all tenant machines share a layer 2
broadcast domain, which may be undesirable in larger clouds. This may be an
acceptable solution for Services in some cases though.

.. _teapot-load-balancing-metallb-l3-management:

MetalLB (Layer 3) on management cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The layer 3 form of MetalLB_ load balancing provides true load balancing, but
requires control over the network hardware in the form of advertising
:abbr:`ECMP (Equal Cost Multiple Path)` routes via BGP. (This also places
additional `requirements
<https://metallb.universe.tf/concepts/bgp/#limitations>`_ on the network
hardware.) Since tenant clusters are not trusted to do this, it would have to
run in the management cluster. There would need to be an API in the management
cluster to vet requests and pass them on to MetalLB, and a
cloud-provider-teapot plugin that tenants could optionally install to connect
to it.

This form of load balancing does not support offloading TLS termination either.

.. _teapot-load-balancing-metallb-l3-tenant:

MetalLB (Layer 3) on tenant cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

While the network cannot trust BGP announcements from tenants, in principle the
management cluster could have a component, perhaps based on `ExaBGP
<https://github.com/Exa-Networks/exabgp#readme>`_, that listens to such
announcements on the tenant V(x)LANs, drops any that refer to networks not
allocated to the tenant, and rebroadcasts the legitimate ones to the network
hardware.

This would allow tenant networks to choose to make use of MetalLB in its Layer
3 mode, providing actual traffic balancing as well as making it possible to
split tenant machines amongst separate L2 broadcast domains. It would also
allow tenants to choose among a much wider range of :doc:`CNI plugins
<./networking>`, many of which also rely on BGP announcements.

This form of load balancing still does not support offloading TLS termination.

.. _teapot-load-balancing-ovn:

Build a new OVN-based load balancer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One drawback of MetalLB is that it is not compatible with using OVN as the
network overlay. This is unfortunate, as OVN is one of the most popular network
overlays used with OpenStack, and thus might be a common choice for those
wanting to integrate workloads running in OpenStack and Kubernetes together.

A new OVN-based network load balancer in the vein of MetalLB might provide more
options for this group.

.. _teapot-load-balancing-ingress-api:

Build a new API using Ingress resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A new API in the management cluster would receive requests in a form similar to
an Ingress resource, sanitise them, and then proxy them to an Ingress
controller running in the management cluster (or some other
centrally-controlled cluster). In fact, it is possible the 'API' could be as
simple as using the existing Ingress API in a namespace with a validating
webhook.

The most challenging part of this would be coaxing the Ingress controllers on
the load balancing cluster to target services in a different cluster (the
tenant cluster). Most likely we would have to sync the EndpointSlices from the
tenant cluster into the load balancing cluster.

In all likelihood when using a software-based Ingress controller running in a
load balancing cluster, a network load balancer would also be used on that
cluster to ensure high-availability of the load balancers themselves. Examples
include MetalLB and `kube-keepalived-vip
<https://github.com/aledbf/kube-keepalived-vip>`_ (which uses :abbr:`VRRP
(Virtual Router Redundancy Protocol)` to ensure high availability). This
component would need to be integrated with :ref:`public IP assignment
<teapot-networking-external>`.

There are already controllers for several types of software load balancers (the
nginx controller is even officially supported by the Kubernetes project), as
well as multiple hardware load balancers. This includes an existing Octavia
Ingress controller in cloud-provider-openstack_, which would be useful for
:doc:`integrating with OpenStack clouds <openstack-integration>`. The ecosystem
around this API is likely to have continued growth. This is also likely to be
the site of future innovation around configuration of network hardware, such as
hardware firewalls.

In general, Ingress controllers are not expected to support non-HTTP(S)
protocols, so it's not necessarily possible to implement the |LoadBalancer|
Service type with an arbitrary plugin. However, the nginx Ingress controller
has support for arbitrary `TCP and UDP services
<https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/>`_,
so the API would be able to provide for either type.

Unlike the network load balancer options, this form of load balancing would be
able to terminate TLS connections.

.. _teapot-load-balancing-custom-api:

Build a new custom API
~~~~~~~~~~~~~~~~~~~~~~

A new service running on the management cluster would provide an API through
which tenants could request a load balancer. An implementation of this API
would provide a pure-software load balancer running in containers in the
management cluster (or some other centrally-controlled cluster). As in the case
of an Ingress-based controller, a network load balancer would likely be used to
provide high-availability of the load balancers.

The API would be designed such that alternate implementations of the controller
could be created for various load balancing hardware. Ideally one would take
the form of a shim to the existing cloud-provider API for load balancers, so
that existing plugins could be used. This would include
cloud-provider-openstack, for the case where Teapot is installed alongside an
OpenStack cloud allowing it to make use of Octavia.

Unlike the network load balancer options, this form of load balancing would be
able to terminate TLS connections.

This option seems to be strictly inferior to using Ingress controllers on the
load balancing cluster to implement an API, assuming both options prove
feasible.

.. _teapot-load-balancing-ingress-controller:

Build a new Ingress controller
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the event that we build a new API in the management cluster, a Teapot
Ingress controller would proxy requests for an Ingress to it. This controller
would likely be responsible for syncing the EndpointSlices to the API as well.

.. _teapot-load-balancing-cloud-provider:

Build a new cloud-provider
~~~~~~~~~~~~~~~~~~~~~~~~~~

In the event that we build a new API in the management cluster, a
cloud-provider-teapot plugin that tenants could optionally install would allow
them to make use of the API in the management cluster to configure Services of
type |LoadBalancer|.

While helpful to increase portability of applications between clouds, this is a
much lower priority than building an Ingress controller. Tenants can always
choose to use Layer 2 MetalLB for their |LoadBalancer| Services instead.

.. _teapot-load-balancing-octavia:

OpenStack Octavia
~~~~~~~~~~~~~~~~~

On paper, Octavia_ provides exactly what we want: a multi-tenant abstraction
layer over hardware load balancer APIs, with a software-based driver for those
wanting a pure-software solution.

In practice, however, there is only one driver for a hardware load balancer
(along with a couple of other out-of-tree drivers), and an Ingress controller
for that hardware also exists. More drivers existed for the earlier Neutron
LBaaS v2 API, but some vendors had largely moved on to Kubernetes by the time
the Neutron API was replaced by Octavia.

The pure-software driver (Amphora) itself supports provider plugins for its
compute and network. However the only currently available providers are for
OpenStack Nova and OpenStack Neutron. Nova will not be present in Teapot. Since
we want to make use of Neutron only as a replaceable implementation detail --
if at all -- Teapot cannot allow other components of the system to become
dependent on it. Additional providers would have to be written in order to use
Octavia in Teapot.

Another possibility is integration in the other direction -- using a
Kubernetes-based service as a driver for Octavia when Teapot is
:doc:`co-installed with an OpenStack cloud <openstack-integration>`.

.. |LoadBalancer| replace:: ``LoadBalancer``

.. _Service: https://kubernetes.io/docs/concepts/services-networking/service/
.. _LoadBalancer: https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer
.. _Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
.. _cloud-provider-openstack: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-octavia-ingress-controller.md#readme
.. _MetalLB: https://metallb.universe.tf/
.. _Octavia: https://docs.openstack.org/octavia/
