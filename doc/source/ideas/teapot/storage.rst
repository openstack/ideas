Teapot Storage
==============

Project Teapot should have the ability to optionally provide multi-tenant
access to shared file, block, and/or object storage. Shared file and block
storage capabilities are not currently available to Kubernetes users except
through the cloud providers.

Tenants can always choose to use hyperconverged storage -- that is to say, both
compute and storage workloads on the same hosts -- without involvement or
permission from Teapot. (For example, by using Rook_.) However, this means that
compute and storage cannot be scaled independently; they are tightly coupled.
Tenants with disproportionately large amounts of data but modest compute needs
(and sometimes vice-versa) would not be served efficiently. Hyperconverged
storage also usually makes sense only for clusters that are essentially fixed.
Changing the size of the cluster results in rebalancing of storage, so it is
not suitable for workloads that vary greatly over time (for instance, training
of machine learning models).

To efficiently run hyperconverged storage also requires a somewhat specialised
choice of servers. Particularly in a large cloud where different tenants have
different storage requirements, it might be cheaper to provide a centralised
storage cluster and thus require either fewer variants or less specialisation
of server hardware.

For all of these reasons, a shared storage pool is needed to take full
advantage of the highly dynamic environment offered by a cloud like Teapot.

Providing multi-tenant access to shared file and block storage allows the cloud
provider to use a dedicated storage network (such as a :abbr:`SAN (Storage Area
Network)`). Many potential users may already have something like this. Having
the storage centralised also makes it easier and more efficient to share large
amounts of data between tenants when required (since traffic can be confined to
the same :ref:`storage network <teapot-networking-storage>` rather than
traversing the public network).

Applications can use object storage anywhere (including outside clouds), but to
minimise network bandwith, it will often be better to have it nearby. Should
the `proposal to add Object Bucket Provisioning
<https://github.com/kubernetes/enhancements/pull/1383>`_ to Kubernetes
eventuate, there will also be advantage in have object storage as part of the
local cloud, using the same authentication mechanism.

Implementation Options
----------------------

OpenStack already provides robust, mature implementations of multi-tenant
shared storage that are accessible from Kubernetes. The main task would be to
integrate them into the system and simplify deployment. These services would
run in either the management cluster or a separate (but still
centrally-managed) storage cluster.

.. _teapot-storage-manila:

OpenStack Manila
~~~~~~~~~~~~~~~~

Manila_ is the most natural fit for Kubernetes because it provides 'RWX'
(Read/Write Many) persistent storage, which is often needed to avoid downtime
when pods are upgraded or rescheduled to different nodes as well as for
applications where multiple pods are writing to the same filesystem in
parallel.

Manila's architecture is relatively simple already. It would be helpful if the
dependency on RabbitMQ could be removed (to be replaced with e.g. json-rpc in
the same way that Ironic has in Metal³), but this would require more
investigation. An Operator for deploying and managing Manila on Kubernetes is
under development.

A :abbr:`CSI (Container Storage Interface)` plugin for Manila already exists in
cloud-provider-openstack_.

.. _teapot-storage-cinder:

OpenStack Cinder
~~~~~~~~~~~~~~~~

Cinder_ is more limited than Manila in the sense that it can provide only 'RWO'
(Read/Write One) access to persistent storage for most applications.
(Kubernetes volume mounts are generally file-based -- Kubernetes creates its
own file system on block devices if none is present.) However, Kubernetes does
now support raw block storage volumes, which *do* support 'RWX' mode for
applications that can work with raw block offsets. KubeVirt in particular is
expected to make use of raw block mode persistent volumes for backing virtual
machines, so this is likely to be a common use case.

Much of the complexity in Cinder is linked to the need to provide agents
running on Nova compute hosts. Since Teapot is a baremetal-only service, only
the parts of Cinder needed to provide storage to Ironic servers are required.
Unfortunately, Cinder is quite heavily dependent on RabbitMQ. However, there
may be scope for simplification through further work with the Cinder community.
The remaining portions of Cinder are architecturally very similar to Manila, so
similar results could be expected.

Cinder has a dependency on Barbican for supporting encrypted volumes. Encrypted
volume support is not required but would be nice to have. This is another
reason to use :ref:`Barbican <teapot-key-management-barbican>`. It would be
nice to think that we could adapt Cinder to be able to use Kubernetes Secrets
instead (perhaps via another key manager back-end to Castellan), but that
doesn't actually provide the :doc:`level of security you would hope for
<key-management>` without Barbican or an equivalent anyway.

A :abbr:`CSI (Container Storage Interface)` plugin for Cinder already exists in
cloud-provider-openstack_.

Ember_ is an alternative CSI plugin that makes use of lib-cinder, rather than
all of Cinder. This allows Cinder's hardware drivers to be used directly from
Kubernetes while eliminating a lot of overhead. However, some of the overhead
that is eliminated is the API that enforces multi-tenancy. Therefore, Ember is
not an option for this particular use case.

.. _teapot-storage-swift:

OpenStack Swift
~~~~~~~~~~~~~~~

Swift_ is a very mature object storage system, with both a native API and the
ability to emulate Amazon S3. It supports :ref:`Keystone <teapot-idm-keystone>`
authentication. It has a relatively simple architecture that should make it
straightforward to deploy on top of Kubernetes.

.. _teapot-storage-radosgw:

Ceph Object Gateway
~~~~~~~~~~~~~~~~~~~

RadosGW_ is a service to provide an object storage interface backed by Ceph,
with two APIs that are compatible with large subsets of Swift and Amazon S3,
respectively. It can use either :ref:`Keystone <teapot-idm-keystone>` or
:ref:`Keycloak <teapot-idm-keycloak>` for authentication. It can be installed
and managed using the Rook_ operator.


.. _Rook: https://rook.io/
.. _cloud-provider-openstack: https://github.com/kubernetes/cloud-provider-openstack#readme
.. _Manila: https://docs.openstack.org/manila/latest/
.. _Cinder: https://docs.openstack.org/cinder/latest/
.. _Ember: https://ember-csi.io/
.. _Swift: https://docs.openstack.org/swift/latest/
.. _RadosGW: https://docs.ceph.com/docs/master/radosgw/
