Converting Scrapers to pupa (continued)
=======================================

Picking up where we left off in :doc:`pupa-conversion`, we'll now look at how to convert committee, bill, vote, and event scrapers from billy to pupa.

Converting Committees
---------------------

There may not be a committee scraper, but good news if there is- it is one of the easier ones to convert.

In pupa committees are just a type of ``Organization``, which we dealt with before when filling out our metadata.

Here are the steps:

1) Update imports and class definition

    ::

        # old
        from billy.scrape.committees import CommitteeScraper, Committee

        # new
        from pupa.scrape import Scraper, Organization

    And::

        # old
        class NCCommitteeScraper(CommitteeScraper):
            jurisdiction = 'nc'

        # new
        class NCCommitteeScraper(Scraper):

2) Add new class name to metadata:

    In ``openstates/nc/__init__.py`` we add::

        from .committees import NCCommitteeScraper

    And add it to the scrapers config::

        scrapers = {
            'people': NCPersonScraper,
            'committees': NCCommitteeScraper,
        }

3) Update ``scrape()`` method:

    As we saw with the legislator scraper, it should no longer take ``term``, and ``chambers`` should become optional.

    You can `check the example diff <https://github.com/openstates/openstates/commit/2b7536bf3aa7ab94d417b24bb27db0a3aaf16bb5#diff-ef744b16368b99cdd23e4c1bd29bd76aR45>`_ for onehow it was done when converting the NC scraper.

    Additionally, it should be converted to ``yield`` objects instead of calling ``save_committee``.

    .. note:: Recall that as before, if this method dispatches to others, it should call them with ``yield from`` and they should be converted to
         yield as needed.

4) Update ``Committee`` references to ``Organization``:

    The old constructor was ``Committee(chamber, committee, subcommittee)``.  Now constructing a committee ``Organization`` looks like::

        committee = Organization(name, chamber=chamber, classification='committee')

    If you're dealing with a subcommittee you need to pass ``parent_id``.  ``parent_id`` can be:

        * another ``Organization`` instance
        * a dictionary like ``{'name': 'Appropriations', 'classification': 'lower'}`` which will find the House Appropriations committee at import time

        .. TODO: ^this is sort of a weird edge case, and could probably be handled a lot better in pupa

    Things to change:

        * the ``.committee`` attribute is now ``.name``
        * if you were manually accessing ``committee['members']`` for any reason that information is now in ``_related``  (A common use case is checking if any members were saved: `like here <https://github.com/openstates/openstates/commit/2b7536bf3aa7ab94d417b24bb27db0a3aaf16bb5#diff-ef744b16368b99cdd23e4c1bd29bd76aL58>`_.

    But for the most part that's it.

    **example diff:** `NC committees conversion <https://github.com/openstates/openstates/commit/2b7536bf3aa7ab94d417b24bb27db0a3aaf16bb5?w=1>`_

5) Running your committee scraper

    Just like before, we should now be able to run our scraper::

        $ docker-compose run scrape nc committees

    This will just run our new work, among the output will be::

        nc (scrape)
          committees: {}
        committees scrape:
          duration:  0:00:06.125869
          objects:
            membership: 1114
            organization: 59

    Check that the organization number seems in line with what you expected.

    And at the end the billy output should match::

        02:42:17 INFO billy: imported 59 committee files

    At this point your committee scraper is most likely ready to go.  Be sure to update your PR and let a reviewer know.


Converting Bills
----------------

The bill scraper is one of the more complex scrapers, but fixing it still follows the same basic principles:

1) Update imports and class definition

    ::

        # old
        from billy.scrape.bills import BillScraper, Bill

        # new
        from pupa.scrape import Scraper, Bill

    and::

        # old
        class NCBillScraper(BillScraper):
            jurisdiction = 'nc'

        # new
        class NCBillScraper(Scraper):

2) Just like we've done before, add the new class name to metadata. (see committees if you need an example)

3) Update ``scrape()`` method:

    The billy scrape method looked like: ``scrape(session, chambers)`` and required both parameters.

    We again need it to scrape the latest sessions bills by default, we can change it to look something like::

        def scrape(self, session=None, chamber=None):
            if not session:
                session = self.latest_session()
                self.info('no session specified, using %s', session)

            chambers = [chamber] if chamber else ['upper', 'lower']
            for chamber in chambers:
                yield from self.scrape_chamber(chamber, session)

4) Update usage of ``Bill`` and its methods:

    There are a lot of small changes here, it is likely easiest to list the examples:

    the constructor::

        # old
        Bill(session, chamber, bill_id, title, type=bill_type)

        # new
        Bill(bill_id, legislative_session=session, chamber=chamber,
             title=title, classification=bill_type)


    Adding versions and documents::

        # old
        bill.add_version(version_name, version_url, mimetype='text/html')

        # new
        bill.add_version_link(version_name, version_url, media_type='text/html')

        # documents would be add_document_link

    .. note:: If there is an on_duplicate param, most likely you'll want to replace it with on_duplicate='ignore', but it may be worth discussion on the ticket.

    Adding sponsors::

        # old
        bill.add_sponsor(spon_type, name, chamber=chamber)

        # new
        bill.add_sponsorship(name, classification=spon_type, 'person',
                             primary=is_primary)
        # if the scraper is aware of committee sponsors you should pass
        # 'organization' for those

    Adding actions::

        # old
        bill.add_action(actor, action, act_date, type=atype)

        # new
        bill.add_action(action, act_date, chamber=actor, classification=atype)
        # act_date should be formatted YYYY-MM-DD

    Adding votes::

        # old
        bill.add_vote(vote)

        # new
        bill.add_vote_event(vote)

    see :ref:`converting-votes` for details on converting a ``Vote`` into a ``VoteEvent``

    .. TODO: add_companion?

5) Fix action categorization:

    If you try to run now you'll get an error that the action types aren't validating.

    The `billy action types <http://docs.openstates.org/en/latest/policies/categorization.html#action-types>`_ have been normalized in Open Civic Data,
    and the new types are `documented there <http://docs.opencivicdata.org/en/latest/scrape/bills.html>`_.

    To ease this transition, you can run::

        $ ./scripts/convert-actions.sh openstates/nc/bills.py

    And it will do an in-place conversion of the action classifications.

    **Be sure to have your work checked-in prior to running on the file in case it does anything weird.**

    You'll also want to remove any categorization of actions as 'other', simply opting for ``None`` instead.

    At this point your bill scraper should be ready to go.

    **example diff:** `NC bill conversion <https://github.com/openstates/openstates/commit/f8cc29b>`_


.. _converting-votes:

Converting Votes
----------------

Votes are a relatively easy process.  There are two major changes:

* They are now called ``VoteEvent``.
* Instead of using 'other' for all votes that aren't a 'yes' or 'no', types like 'excused', 'absent' and 'not voting' have been added.

1) Update imports and class definition


    ::
        # old
        from billy.scrape.bills import VoteScraper, Vote

        # new
        from pupa.scrape import Scraper, VoteEvent

    and::

        # old
        class NCVoteScraper(VoteScraper):
            jurisdiction = 'nc'

        # new
        class NCVoteScraper(Scraper):

2) Just like we've done before, add the new class name to metadata. (see committees if you need an example)

3) Update ``scrape()`` method:

    The logic here will be almost identical to what you did in the bill scraper.

    We need it to scrape the latest sessions votes by default.

4) Update usage of ``Vote`` to ``VoteEvent``:

    The old ``Vote`` constructor took a ton of parameters::

        Vote(chamber, date, motion, passed,
             yes_count, no_count, other_count, type='other', **kwargs)

        # often there'd be additional parameters:

        Vote(chamber, date, motion, passed,
             yes_count, no_count, other_count, type='other',
             bill_id=bill_id, bill_chamber=bill_chamber, session=session,
             )

    Be careful since many of the older scrapers pass these in by position alone, it'd be easy to mistake the old order when converting.

    ``VoteEvent`` requires all parameters to be passed by keyword::

        VoteEvent(chamber=chamber,
                  start_date='2017-03-04',
                  motion_text=motion,
                  result='pass' if passed else 'fail',
                  classification='passage',     # can also be 'other'

                  # required if not being passed to bill.add_vote
                  legislative_session=session,
                  bill=bill_id,
                  bill_chamber=bill_chamber)

    You'll notice that in the instantiation of the class we didn't pass
    yes_count, no_count, other_count.  Instead we'll set these using ``set_count``::

        vote.set_count('yes', yes_count)
        vote.set_count('no', no_count)

        # if possible, we'll split other out into the various values given
        vote.set_count('absent', absent_count)
        vote.set_count('not voting', not_voting_count)


    Individual legislator's votes are added to the ``VoteEvent`` in the same way they were, the only exception being ``.other``::

        # these haven't changed
        vote.yes(yes_voter_name)
        vote.no(no_voter_name)

        # old-style
        vote.other(other_voter_name)
        # new-style
        vote.vote('not voting', not_voting_name)
        vote.vote('absent', absentee_name)


    Our example state of NC was a bit more complex to change due to some unusual behavior, but nonetheless here's the **example diff:** `NC vote conversion <https://github.com/openstates/openstates/commit/61aaa4eb>`_

        
