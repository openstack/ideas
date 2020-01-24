Teapot Compute
==============

Project Teapot is conceived as an exclusively bare-metal compute service for
Kubernetes clusters. Providing bare-metal compute workers to tenants allows
them to make their own decisions about how they make use of virtualisation. For
example, Tenants can choose to use a container hypervisor (such as Kata_) to
further sandbox applications, traditional VMs (such as those managed by
KubeVirt_ or `OpenStack Nova`_), *or both* `side-by-side
<https://kubernetes.io/docs/concepts/containers/runtime-class/>`_ in the same
cluster. Furthermore, it allows users to manage all components of an
application -- both those that run in containers and those that need a
traditional VM -- from the same Kubernetes control plane (using KubeVirt).
Finally, it eliminates the complexity of needing to virtualise access to
specialist hardware such as :abbr:`GPGPU (general-purpose GPU)`\ s or FPGAs,
while still allowing the capability to be used by different tenants at
different times.

However, the *master* nodes of tenant cluster will run in containers on the
management cluster (or some other centrally-managed cluster). This makes it
easy and cost-effective to provide high availability of cluster control planes,
by not sacrificing large numbers of hosts to this purpose or requiring
workloads to run on master nodes. It also makes it possible to optionally
operate Teapot as a fully-managed Kubernetes service. Finally, it makes it
relatively cheap to scale a cluster to zero when it has nothing to do, for
example if it is only used for batch jobs, without requiring it to be recreated
from scratch each time. Since the management cluster also runs on bare metal,
the tenant pods could also be isolated from each other and from the rest of the
system using Kata, in addition to regular security policies.

.. _teapot-compute-metal3:

Metal³
------

Provisioning of bare-metal servers will use `Metal³`_.

The baremetal-operator from Metal³ provides a Kubernetes-native interface over
a simplified `OpenStack Ironic`_ deployment. In this configuration, Ironic runs
standalone (i.e. it does not use Keystone authentication). All communication
between components occurs inside of a pod. RabbitMQ has been replaced by
json-rpc. Ironic state is maintained in a database, but the database can run on
ephemeral storage -- the Kubernetes custom resource (BareMetalHost) is the
source of truth.

The baremetal-operator will run only in the management cluster (or some other
centrally managed cluster) because it requires access to both the :abbr:`BMC
(Baseboard Management Controller)`\ s' network (as well as the
:ref:`provisioning network <teapot-networking-provisioning>`) and the
authentication credentials for the BMCs.

.. _teapot-compute-cluster-api:

Cluster API
-----------

The baremetal-operator can be integrated with the Kubernetes Cluster Lifecycle
SIG's `Cluster API`_ via another Metal³ component, the
cluster-api-provider-baremetal. This contains a BareMetalMachine controller
that implements the Machine abstraction using a BareMetalHost. (Airship_ 2.0 is
also slated to use Metal³ and the Cluster API to manage cluster provisioning,
so this mechanism could be extended to deploy fully-configured clusters with
Airship as well.)

When the Cluster API is used to build standalone clusters, typically a
bootstrap node is created (often using a local VM) to run it in order to create
the permanent cluster members. The Cluster and Machine resources are then
'pivoted' (copied) into the cluster, which continues to manage itself while the
bootstrap node is retired. When used with a centralised cluster manager such as
Teapot, the process is usually similar but can use the management cluster to do
the bootstrapping. Pivoting is optional but usually expected.

Teapot imposes some additional constraints. Because the BareMetalHost objects
must remain in the management cluster, the Machine objects cannot be simply
copied to the tenant cluster and continue to be backed by the BareMetalMachine
controller in its present form.

One option might be to build a machine controller for the tenant cluster that
is backed by a Machine object in another cluster (the management cluster). This
might prove useful for centralised management clusters in general, not just
Teapot. We would have no choice but to name this component
cluster-api-provider-cluster-api.

Cluster API does not yet have support for running the tenant control plane in
containers. Tools like Gardener_ do, but are not yet well integrated with the
Cluster API. However, the Cluster Lifecycle SIG is aware of this use case, and
will likely evolve the Cluster API to make this possible.

.. _teapot-compute-autoscaling:

Autoscaling
-----------

The preferred mechanism in Kubernetes for applications to control the size of
the cluster they run in is the Cluster Autoscaler. There is no separate
interface to this mechanism for applications. If an application is too busy, it
simply requests more or larger pods. When there is no longer sufficient
capacity to schedule all requested pods, the Cluster Autoscaler will scale the
cluster up. Similarly, if there is significant excess capacity not being used
by pods, Cluster Autoscaler will scale the cluster down.

Cluster Autoscaler works using its own cloud-specific plugins. A `plugin that
uses the Cluster API is in progress
<https://github.com/kubernetes/autoscaler/pull/1866>`_, so Teapot could
automatically make use of that provided that the Machine resources were pivoted
into the tenant cluster.

One significant challenge posed by bare-metal is the extremely high latency
involved in provisioning a bare-metal host (15 minutes is not unusual, due in
large part to running hardware tests including checking increasingly massive
amounts of RAM). The situation is even worse when needing to deprovision a host
from one tenant before giving it to another tenant, since that requires
cleaning the local disks, though this extra overhead can be essentially
eliminated if the disk is encrypted (in which case only the keys need be
erased).

.. _teapot-compute-scheduling:

Scheduling
----------

Unlike when operating a standalone bare-metal cluster, when allocating hosts
amongst different clusters it is important to have sophisticated ways of
selecting which hosts are added to which cluster.

An obvious example would be selecting for various hardware traits -- which are
unlikely to be grouped into 'flavours' in the way that Nova does. The optimal
way of doing this would likely include some sort of cost function, so that a
cluster is always allocated the minimum spec machine that meets its
requirements. Another example would be selecting for either affinity or
anti-affinity of hosts, possibly at different (and deployment-specific) levels
of granularity.

Work is underway in Metal³ on a hardware-classification-controller that will
add labels to BareMetalHosts based on selected traits, and the baremetal
actuator can select hosts based on labels. This would be sufficient to perform
flavour-based allocation and affinity, but likely not on its own for
trait-based allocation and anti-affinity.

.. _teapot-compute-reservation:

Reservation and Quota Management
--------------------------------

The design for quota management should recognise the many ways in which it is
used in both private and public clouds. In public clouds utilisation is
controlled by billing; quotas are primarily a tool for *users* to limit their
financial exposure.

In private OpenStack clouds, the implementation of chargeback is rare. A more
common model is that a department will contribute a portion of the capital
budget for a cloud in exchange for a quota -- a model that fits quite well with
Teapot's allocation of entire hosts to tenants.

To best support the private cloud use case, there need to be separate concepts
of a guaranteed minimum reservation and a maximum quota. The sum of minimum
reservations must not exceed the capacity of the cloud (are more complex
requirement than it sounds, since it must take into account selected hardware
traits). Some form of pre-emption is needed, along with a way of prioritising
requests for hosts. Similar concepts exist in many public clouds, in the form
of reserved and spot-rate instances.

The reservation/quota system should have a time component. This allows, for
example, users who have large batch jobs to reserve capacity for them without
tying it up around the clock. (The increasing importance of machine learning
means that once again almost everybody has large batch jobs.) Time-based
reservations can also help mitigate the high latency of moving hosts between
tenants, by allowing some of the demand to be anticipated.


.. _Kata: https://katacontainers.io/
.. _KubeVirt: https://kubevirt.io/
.. _OpenStack Nova: https://docs.openstack.org/nova
.. _Metal³: https://metal3.io/
.. _OpenStack Ironic: https://docs.openstack.org/ironic
.. _Cluster API: https://github.com/kubernetes-sigs/cluster-api#readme
.. _Airship: https://www.airshipit.org/
.. _Gardener: https://gardener.cloud/030-architecture/
