Teapot Identity Management
==========================

Teapot need not, and should not, impose any particular identity management
system for tenant clusters. These are the clusters that applications and
application developers/operators will routinely interact with, and the choice
of identity management providers is completely up to the administrators of
those clusters, or at least the administrator of the Teapot cloud when running
as a fully-managed service.

Identity management in Teapot itself (i.e. the management cluster) is needed
for two different purposes. While not strictly necessary, it would be
advantageous to require only one identity management provider to cover both of
these use cases.

Authenticating From Below
-------------------------

Software running in the tenant clusters needs to authenticate to the cloud to
request resources, such as machines, :doc:`load balancers <load-balancing>`,
:doc:`shared storage <storage>`, :doc:`DNS records <dns>`, and (in future)
managed software services.

Credentials for these purposes should be regularly rotated and narrowly
authorised, to limit both the scope and duration of any compromise.

Authenticating From Above
-------------------------

Real users and sometime software services need to authenticate to the cloud to
create or destroy clusters, manually scale them up or down, request quotas, and
so on.

In many cases, such as most enterprise private clouds, these credentials should
be linked to an external identity management provider. This would allow
auditors of the system to tie physical hardware directly back to corporeal
humans to which it is allocated and the organisational units to which they
belong.

Humans must also have a secure way of delegating privileges to an application
to interact with the cloud in this way -- for example, imagine a CI system that
needs to create an entire test cluster from scratch and destroy it again. This
must not require the user's own credentials to be stored anywhere.

Implementation options
----------------------

.. _teapot-idm-keystone:

OpenStack Keystone
~~~~~~~~~~~~~~~~~~

Keystone_ is currently the only game in town for providing identity management
for OpenStack services that are candidates for being included to provide some
multi-tenant functionality in Teapot, such as :ref:`Manila
<teapot-storage-manila>` and :ref:`Designate <teapot-dns-designate>`. Therefore
using Keystone for all identity management on the management cluster would not
only not increase complexity of the deployment, it would actually minimise it.

An authorisation webhook for Kubernetes that uses Keystone is available in
cloud-provider-openstack_. In general, OAuth seems to be preferred to webhooks
for connecting external identity management systems, but there is at least a
working option.

Keystone supports delegating user authentication
to LDAP, as well as offering its own built-in user management. It can also
federate with other identity providers via the `OpenID Connect`_ or SAML_
protocols. Using Keystone would also make it simpler to run Teapot alongside an
existing OpenStack cloud -- enabling tenants to share services in that cloud,
as well as potentially making Teapot's functionality available behind an
OpenStack-native API (similar to Magnum) for those who want it.

Keystone also features quota management capabilities that could be reused to
manage tenant quotas_. A proof-of-concept for a validating webhook that allows
this to be used for governing Kubernetes resources `exists
<https://github.com/cmurphy/keyhook#readme>`_.

While there are generally significant impedance mismatches between the
Kubernetes and Keystone models of authorisation, Project Teapot is a fresh
start and can prescribe custom policy models that mitigate the mismatch.
(Ongoing changes to default policies will likely smooth over these kinds of
issues in regular OpenStack clouds also.) This may not be so easy when sharing
a Keystone :doc:`with an OpenStack cloud <openstack-integration>` though.

Keystone Application Credentials allow users to create (potentially)
short-lived credentials that an application can use to authenticate without the
need to store the user's own LDAP password (which likely also governs their
access to a wide range of unrelated corporate services) anywhere. Credentials
provided to tenant clusters should be exclusively of this type, limited to the
purpose assigned (e.g. credentials intended for accessing storage can only be
used to access storage), and regularly rotated out and expired.

.. _teapot-idm-dex:

Dex
~~~

Dex_ is an identity management service that uses `OpenID Connect`_ to provide
authentication to Kubernetes. It too supports delegating user authentication to
LDAP, amongst others. This would likely be seen as a more conventional choice
in the Kubernetes community. Dex can store its data using Kubernetes custom
resources, so it is the most lightweight option.

Dex does not support authorisation. However, Keystone supports OpenID Connect
as a federated identity provider, so it could still be used as the
authorisation mechanism (including for OpenStack-derived services such as
Manila) using Dex for authentication. However, this inevitably adds additional
moving parts. In general, Keystone has `difficultly
<https://bugs.launchpad.net/keystone/+bug/1589993>`_ with application
credentials for federated users because it is not immediately notified of
membership revocations, but since both components are under the same control in
this case it would be easier to build some additional integration to keep them
in sync.

.. _teapot-idm-keycloak:

Keycloak
~~~~~~~~

Keycloak_ is a more full-featured identity management service. It would also be
seen in the Kubernetes community as a more conventional choice than Keystone,
although it does not use the Kubernetes API as a data store. Keycloak is
significantly more complex to deploy than Dex. However, a `Kubernetes operator
for Keycloak <https://operatorhub.io/operator/keycloak-operator>`_ now exists,
which should hide much of the complexity.

Keystone could federate to Keycloak as an identity management provider using
either OpenID Connect or SAML.

Theoretically, Keycloak could be used without Keystone if the Keystone
middleware in the services were replaced by some new OpenID Connect middleware.
The architecture of OpenStack is designed to make this at least possible. It
would also require changes to client-side code (most prominently any
cloud-provider-openstack providers that might otherwise be reused), although
there is a chance that they could be contained to a small blast radius around
Gophercloud's `clientconfig module
<https://github.com/gophercloud/utils/tree/master/openstack/clientconfig>`.


.. _Keystone: https://docs.openstack.org/keystone/
.. _OpenID Connect: https://openid.net/connect/
.. _SAML: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
.. _cloud-provider-openstack: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-keystone-webhook-authenticator-and-authorizer.md#readme
.. _quotas: https://docs.openstack.org/keystone/latest/admin/unified-limits.html
.. _Dex: https://github.com/dexidp/dex/#readme
.. _Keycloak: https://www.keycloak.org/
