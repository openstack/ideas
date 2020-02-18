===================
Ideas for OpenStack
===================

About this website
==================

This website is a breeding ground for ideas in OpenStack.
Everyone with an idea can propose it here, even if said idea has been
discussed/proposed in the past.

Keep in mind, the ideas in this repository can be very fresh or very old.
The age of the idea does not matter for your contribution.
Someone wanting to iterate over any idea (even an old one) can do so.
It is recommended to check the history using git first, though, as
the history (linked with ML topics) might reveal why an old idea was not
implemented.

Are the Mailing Lists not enough?
---------------------------------

Mailing lists are great, use them! In fact, we encourage you to share your idea
on the mailing list first. However, there are places where Mailing lists fall
short. For example, the history repeats itself, and people ask the same
questions all over again "Why don't we change the cycle duration?",
"Can we change the PTL role?". One possible reason for this is the lack of
an easy to way to propose, search, browse history, and iterate on ideas.
The ideas repository is an attempt to fix things.

Can/Should I propose an idea here?
----------------------------------

Everyone can propose an idea. You don't need to have special credentials.
The technical committee is responsible for merging your ideas into this
repository, but it stops there. Only one vote will be required, and the only
negative votes you can get are the ones that don't reflect what was said on
the mailing lists. So there is no reason to hold off a patch.

If you have an idea and don't want it to be lost in the memories,
propose it here.

Proposing an idea doesn't mean being in charge of implementing it. You are
just sharing your idea for increasing the collective wisdom.

What is the process for any contributions here?
-----------------------------------------------

The purpose of this system is to be VERY lightweight.
Even though the repository is curated by the technical committee, the process
to propose an idea should be done in minutes.

1. Copy the ``ideas/EXAMPLE`` into ``ideas/<my_awesome_idea_name>.rst``
2. Edit the content of ``ideas/<my_awesome_idea_name>.rst``. The only thing we
   ask (to help figuring out what the idea is about), is to always have at least
   2 paragraphs: 1 explaining the idea itself, 1 explaining why you want it.
   You can add any extra text into your idea, but we need those two.
3. When this is created, ``git checkout -b my_awesome_idea_name &&
   git add . && git commit && git review -f``.
   In your commit message, please ALWAYS mention a conversation on the mailing
   list. In other words, make sure your idea or idea update was discussed,
   and that your commit is representative of a conversation.
   This is necessary because it helps retracing updates with the mailing list,
   when checking the git log.

There is no limit in terms of the amounts of commit for any single idea.

What if my idea is completely crazy?
------------------------------------

There is NO reason to keep that for you, as long as it is relevant for
OpenStack as a whole. Crazy ideas are more than welcome!
On top of that, you might find other crazy people to help you on the way.

I think idea <x> is plain wrong/outdated, how can I fix this?
-------------------------------------------------------------

Just speak about it on the Mailing list, and edit the idea in this repo.

I like this idea, how can I help?
---------------------------------

That's simple: Check ``git blame`` on the current idea. You will find people
interested in the idea. You can then contact them about progress and give a
hand.

Proposed ideas
==============

.. toctree::
   :maxdepth: 2
   :titlesonly:
   :glob:

   ideas/*
