..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Process And Deadlines For Specs
===============================

The Manila spec repo was created for the Newton release and did not get used
very effectively. Many specs did not get reviews, even fewer specs were merged
during the release, and some features with merged specs did not get code review
priority.

In order to ensure that specs do get reviews, and also to help focus the
efforts of the reviewers, I propose adding some official deadlines and rules
for handling of specs.


Problem description
===================

We have a few specific problems, some new in Newton, and some which have grown
worse over time:

* Some specs get minimal reviews, or don't get reviewed at all.

* There is no reason to merge a spec because we didn't attach any meaning to
  the action of merging a spec, so most specs were not merged.

* The core reviewer team has too little resources to review all of the proposed
  features in a release.

* Code reviews are turning up architectural issues which could have been caught
  during a spec review, and the problems are being found very late in the
  release cycle.

* Contributors have very little idea whether their changes will get merged
  until right before feature freeze, because we rarely say "no" to anything
  unless it misses a deadline.


Use Cases
=========

The purpose of the proposed change is to focus the core reviewer team on a
smaller set of features, and to give contributors earlier feedback on their
proposals. Rather than having a lot of things with small chances of success,
we would like to have a few things with large chances of success.


Proposed change
===============

First, I propose defining what it means to merge a spec. Merging a spec during
a release will mean:

* We are happy with the definition for the proposed feature, and it fits with
  the direction for the project and the current release.

* The feature will be targeted at the current release, and will receive code
  review attention and testing.

* The team intends to merge the completed feature before feature freeze, if
  the submitter has completed code and is responsive to feedback.

Note that merging a spec will not guarantee that the code for a feature will
merge only that we will try. More importantly, NOT merging a spec will mean
that we don't intend to work on that feature, and it will be reconsidered for
the following release.

Next, I propose a specs deadline:

* Low priority specs must merge by the milestone-1 date. Specs not deemed
  high priority and not merged by milestone-1 will be taken out of
  consideration for the release, to help focus the team.

* High priority specs must be merged by the milestone-2 date. Any spec not
  merged by the milestone-2 date will be taken out of consideration for the
  release, to help focus the team.

* Specs must be deemed high priority by the team before the milestone-1 date.
  In many cases high priority specs will be decided earlier but the
  milestone-1 will be the final deadline so we can start narrowing our scope
  for a release.

* High priority specs will be designated by adding them to an index file
  called high_priority.rst in the appropriate release directory in the specs
  tree. Modifications to this file will be controlled by gerrit.

Because merging a spec will imply that the team will spend sufficient time
reviewing that feature, the team will need to limit the number of merged specs,
even if specs meet all of the other requirements to be merged. The team will
need to prioritize the proposed features and merge the higher priority ones
first, and stop merging specs when there's no more available review bandwidth.

Because the number of proposed specs can be more than the whole team can
reasonably review in the time before the first milestone, the team should
designate a subset of the specs as "review focus" specs which everyone is
expected to review, while the remaining specs will be reviewed on a volunteer
basis. Review focus specs will be listed on an etherpad agreed to by the core
reviewer team.

Lastly, I propose that we require specs for any kind of change with broad
impacts. It's better to agree on the design before we get too deep into
implementation.

New drivers and changes to drivers should not ever require specs, and thus
driver changes are not affected by these deadlines. The exception would be
a change to a driver that implemented something unusual or controversial
and thus required community discussion.


Decisions
=========

This specs process requires several decisions to be made:

* Review focus designation for specs

* Prioritization for specs, including stack ranking and low/high priority
  designation

* Cut line for merging specs

* Merge decisions for individual specs

It's important for the contributors involved with Manila to have influence
over the direction for the project, so I propose that none of these decisions
be made solely by the PTL, and that decisions should be made as democratically
as practical (with PTL resolving deadlocks).

The whole community should provide input on the decision for which specs to
focus on for reviews, and also the prioritization of specs. We want to work on
things which the largest number of people will find valuable so we will gather
input during weekly meetings and using etherpads about what we should focus on.

Determination of the cut line has to be made by the core reviewer team because
the commitment to review new featutes comes from the core reviewers and the
goal of this change is to avoid overcommitting.

Merging of individual specs should be done as usual with two +2s from core
reviewers from different companies if the spec is low priority, and with +2
from a majority of the core reviewers for high priority specs.

Designation of a spec as "high priority" which changes the merge deadline from
milestone-1 to milestone-2 should be done by agreement by the core reviewer
team, because of the additional review commitment and increased risk such a
decision carries.

The purpose of giving "high priority" specs more time than "low priority"
specs is not to divert attention away from the high priority stuff, but to
make the decision to push stuff out of the release as early as possible. This
process can't make people review specs earlier if they're not inclined to, but
we can reduce the set of specs that need reviewing by drawing a cut line
fairly early, and hopefully that will focus our limited review time more
productively.
