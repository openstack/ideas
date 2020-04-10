Asynchronous communications
===========================

Before explaining what tools can be good for OpenStack, we should clarify
what asynchronous communications mean, and explain the problem with the
current status-quo.

Definition
----------

The *asynchronous* communications are the communications happening between
community members which don't trigger an expectation of an immediate
("instant") answer from other community members.

The current mechanisms for asynchronous communications are for example our
code review system, ask.openstack.org, or our mailing lists.

The problem
-----------

While our code review system brings an easy way to track comments on any
code and find history of discussions and decisions, a code review system is
not a good fit for non-code conversations.

For that we rely on mailing lists. The current mailing list implementation has
a poor user interface, preventing efficient search for example.

Next to this, a mailing list is only an email service. It will not provide
any other tool that our community could need and use in asynchronous
communications. An example is a poll system for a topic: It often happen
that we run informal polls in OpenStack governance entities (like projects),
yet the only way to record those polls is to rely on either an external
service, or to rely on an IRC bot (and need to fallback to a synchronous
meeting).

We should move to a more modern tool for asynchronous communications,
as it would offer new features, be more attractive to interact, and
look more modern to accomodate new contributors.

The proposal
------------

I propose to move our mailing lists to `Discourse`_, like Mozilla.

`Discourse`_ has lots of benefits: a better search, indexing, and
interaction with communication tools. This would allow to refer to synchronous
communications inside the asynchronous tool. For example, we could
have conversations with `matrix`_ inside `Discourse`_.
(https://www.discourse.org/plugins/chat-integration.html)

Next to this, we could generate more granular data on the engagement on certain
topics, compared to the mailing lists, as it is already part of Discourse.

Fun note: This would also allow us to remove this "ideas" framework
git repository.

Downsides
---------

Discourse if far less attractive for "email only" workflows.
While it is possible to simply "Reply via email", and mark all the conversations
as read when they are sent through email, creating a new topic is a little
bit harder, as a user cannot simply send to the generic address. Instead,
the writer should find the unique email for a new topic.
See also Mozilla's `new Discourse topic via email procedure`_.

A change in the tooling could lead to a split in the community,
for people not wanting to change. There is of course an
`xkcd for that`_.

.. _new Discourse topic via email procedure: https://discourse.mozilla.org/t/how-do-i-use-discourse-via-email/15279#create-topics-via-email
.. _xkcd for that: https://xkcd.com/1782/
.. _Discourse: https://www.discourse.org/
.. _matrix: https://matrix.org/
