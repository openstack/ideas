Synchronous and pseudo-synchronous communications
=================================================

Before explaining what tools can be good for OpenStack, we should clarify
what (pseudo-)synchronous communications mean, and explain the problem with the
current status-quo.

Definition
----------

The *synchronous* and *pseudo-synchronous* communications are the
happening between our community members, for which people
expect a relatively quick answer. The relative notion
depends on the context. For example, on a idle IRC channel, one might not
be expected to answer instantly, while instant present is expected during
synchronous IRC meetings or video conferences.

The current mechanisms for (pseudo-)synchronous communications are for
example is OpenDev's meetpad service and our freenode IRC channels.

The problem
-----------

With the community being more distributed, it is probably easier to deal
with the distribution by relying on more asynchronous tools.
Yet, our processes and asynchronous tools limitations make us use more
synchronous communications, which limits the scalability of the community.
Some people will always feel left alone in the current ways.

Next to this, IRC seems alien to the "new generation of developers", which
prefer *Slack* or *Matrix*: Those record conversations when you are
disconnected for example.

The proposal
------------

I propose to move our IRC communication by moving to `matrix`_ (and optionally
a front-end/web client, like *Element*).

I believe using it would increase attractiveness of our synchronous
communications. Additionally, this could integrate well with a more
modern asynchronous tooling like `Discourse`_.

Mozilla has also chosen `matrix`_.

People using IRC could easily adopt matrix, as a IRC bridge exists.

For new joiners, a web interface would be a welcomed change,
as communication can be done from the browser, like *Slack*.

Downsides
---------

This is a disruptive change, like for the asynchronous communications.
This could lead to a split in the community, for people not wanting to change.
https://xkcd.com/1782/


.. _matrix: https://matrix.org/
.. _Discourse: https://www.discourse.org/
