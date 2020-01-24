Teapot Key Management
=====================

Kubernetes offers the Secret resource for storing secrets needed by
applications. This is an improvement on storing them in the applications'
source code, but unfortunately by default Secrets are not encrypted at rest,
but simply stored in etcd in plaintext. An EncryptionConfiguration_ resource
can be used to ensure the Secrets are encrypted before storing them, but in
most cases the keys used to encrypt the data are themselves stored in etcd in
plaintext, alongside the encrypted data.

This can be avoided by using a `Key Management Service provider`_ plugin. In
this case the encryption keys for each Secret are themselves encrypted, and can
only be decrypted using a master key stored in the key management service
(which may be a hardware security module). All extant KMS providers appear to
be for cloud services; there are no baremetal options.

Since the KMS provider is necessary to provide effective encryption at rest and
is the *de facto* responsibility of the cloud, it would be desirable for Teapot
to support it. The implementation should be able to make use of :abbr:`HSM
(Hardware Security Module)`\ s, but also be able to work with a pure-software
solution.

Implementation Options
----------------------

.. _teapot-key-management-barbican:

OpenStack Barbican
~~~~~~~~~~~~~~~~~~

Barbican_ provides exactly the thing we want. It `provides
<https://docs.openstack.org/barbican/latest/install/barbican-backend.html>`_ an
abstraction over HSMs as well as software implementations using Dogtag_ (which
can itself store its master keys either in software or in an HSM) or Vault_,
along with another that simply stores its master key in the config file.

Like other OpenStack services, Barbican uses Keystone for :doc:`authentication
<idm>`. A :abbr:`KMS (Key Management Service)` provider for Barbican already
exists in cloud-provider-openstack_. This could be used in both the management
cluster and in tenant clusters.

Barbican's architecture is relatively simple, although it does rely on RabbitMQ
for communication between the API and the workers. This should be easy to
replace with something like json-rpc as was done for Ironic in Metal³ to
simplify the deployment.

Storing keys in software on a dynamic system like Kubernetes presents
challenges. It might be necessary to use a host volume on the master nodes to
store master keys when no HSM is available. Ultimately the most secure solution
is to use a HSM.

.. _teapot-key-management-secrets:

Write a new KMS plugin using Secrets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Writing a KMS provider plugin is very straightforward. We could write one that
just uses a Secret stored in the management cluster as the master key.

However, this could not be used to encrypt Secrets at rest in the management
cluster itself.


.. _EncryptionConfiguration: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
.. _Key Management Service provider: https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/
.. _Barbican: https://docs.openstack.org/barbican/latest/
.. _Dogtag: https://www.dogtagpki.org/wiki/PKI_Main_Page
.. _Vault: https://www.vaultproject.io/
.. _cloud-provider-openstack: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-barbican-kms-plugin.md#readme
