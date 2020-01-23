# Qiskit RFCs
The purpose of a Qiskit Request for Comments (RFC) is to communicate and engage
with the community in the development and direction of Qiskit. RFCs enable
engineering and research stakeholders to communicate large design changes.

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the Qiskit community and
the sub-teams.

# When Should You Write an RFC?
RFC's should be reserved for 'substantial' changes to the Qiskit meta-package,
its members and the RFC process itself. By this, we mean changes
where the implementation path is not immediately clear and needs to be
deconstructed by the larger Qiskit team.

Questions you might ask yourself:
- Will the implementation involve many developers?
- Will the implementation span across multiple points in the Qiskit stack?
- Will the changes cause ramifications for the average user?
- Will the changes require collaboration with outside sources?

# Process
- Fork the [rfcs repository](https://github.com/Qiskit/rfcs)
- Copy the template [0000-template.md](0000-template.md) to
`####-rfc-title.md`, do not yet assign a number. If the RFC requires additional
files, it may be placed in the `text` folder with the name `####-rfc-title`.
- Fill in the template with your RFC. Be thorough and convincing, use proper
grammar and technical language where appropriate. The aim of an RFC is to
convey both a change and a vision for the future it will enable, you must
convince the larger Qiskit team that it is valuable.
- Submit a pull request to the Qiskit meta-package titled "[RFC] RFC Title"
- Each RFC will be labeled with the relevant packages, so that the respective
maintainers of the packages may be notified of the RFC.
- The RFC will be triaged and if it is of sufficient quality a
*review committee* will be formed by assigning the PR to a group of
*committee members* who are each maintainer(s) of the relevant Qiskit packages.
Committee members are responsible for moderating the development of the RFC and
acceptance or closure of the RFC. It is expected that RFC should fail in the
early, rather than later stages of the development cycle. If the RFC author is
themselves a maintainer of one of the relevant packages, they should not be a
committee member for their own RFC.
- Interested parties should discuss and modify the RFC within the pull-request.
Efforts should be made to summarize offline discussions within the PR. The aim
is to capture the outcome of discussion within the RFC, and the flow of
development within the PR.
- The RFC will go through many iterations at this stage, *do not* squash/rebase
the RFC commits. The aim is to capture the history of the document.
- When the RFC has satisfied a committee member, they should review and approve
the PR. If it is not progressing satisfactorily, or supported by the review
committee it may be closed at any time. This may be petitioned by reopening the
PR, along with a potential request for a new review committee.
- Upon approval by all review committee members, the RFC will be assigned a
number of `max(rfc_####) + 1`, the filename will be updated to reflect this and
the author list should be validated. Note that as Qiskit is still undergoing
rapid-development there is no required grace period between acceptance and
merger, as the project matures this is expected to change.
- The RFC will then be merged by one of the committee members.
- After acceptance, the implementation of the contents of the RFC may proceed.

## The RFC life-cycle

Once an RFC becomes "active" then authors may implement it and submit the
feature as a pull request to the relevant Qiskit repos. Being "active" is not
a rubber stamp, and in particular still does not mean the feature will
ultimately be merged; it does mean that in principle all the major stakeholders
have agreed to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a Qiskit developer has been assigned the task of
implementing the feature. While it is not *necessary* that the author of the
RFC also write the implementation, it is by far the most effective way to see
an RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We
strive to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect every
merged RFC to actually reflect what the end result will be at the time of the
next major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC. Exactly what counts
as a "very minor change" is up to the sub-team to decide; check
[Sub-package guidelines] for more details.

## Sub-package guidelines
For more details on RFCs for specific Qiskit projects see the guidelines for
each sub-package:

### Qiskit Terra

### Qiskit Aer

### Qiskit Ignis

### Qiskit Aqua

RfCs which are primarily or substantially relevant to Aqua must be reviewed in Aqua's weekly _design review_ meeting, with few exceptions. The rules and process of design review are as follows:
* **Initiation and Scheduling:** Once sufficent discussion has occured and the RfC has settled into a near-complete form, the author(s) should note on the PR that the RfC is ready for review. At this point, the Aqua committee member assigned to the RfC will contact the RfC author(s) to schedule a design review slot, determine who on the broader Qiskit team should be invited to the review, and send a meeting invitation to the review attendees. At least 48 hours before design review, a reminder email will be sent to the attendees, including the RfC. Any changes to the RfC during that 48 hour window will be assumed not to have been seen by the review attendees.
* **Design Review Meeting**
    * The Aqua committee member will conduct the meeting, including tabling larger-topics to keep the discussion on track.
    * Attendees must read the design doc ahead of time to participate in the discussion. If they do not, they are choosing to be a listener only. The discussion will begin with each attendee sharing whether they've read the RfC, and their high-level impressions and/or concerns (about 2 minutes each).
    * Author(s) will then spend 15-20 minutes talking about their intentions with the doc and the open questions they need answered by the review participants.
    * Finally, each participant will give a recommendation:
        * Approve - begin implementation
        * Conditionally approve - Begin implementation once some small questions are answered offline
        * Rereview - review again after more information is collected before implementing
        * Downscope - Revise the project's scope or direction and rereview before implementing
    * The Aqua committee member will decide the status of the RfC, or how to proceed, based on the recommendation tally.
* **Final Approval** Following review, the discussion should be posted to the PR, including any action items. If approved or conditionally approved, the Aqua committee member will review the final document or changes, and approve and merge when appropriate. If designated for rereview or downscoping, the author(s) should address the concerns of the review participants, and re-initiate the design review process once ready.

### Qiskit IBMQ Provider

## RFC Postponement

Some RFC pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An RFC closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. This normally means
a topic for discussion after the Qiskit 1.0 release. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask members of the relevant sub-team.

Usually an RFC pull request marked as "postponed" has already passed an
informal first round of evaluation, namely the round of "do we think we would
ever possibly consider making this change, as outlined in the RFC pull request,
or some semi-obvious variation of it." (When the answer to the latter question
is "no", then the appropriate response is to close the RFC, not postpone it.)

# Contributors
An *RFC author* write and champions and RFC through the process.

A *community member* provides feedback on an RFC either as a PR comment, or an
edit to the RFC.

A *review committee* is the group *committee members*, all of which are RFC PR
assignees and are maintainers of one or many of the Qiskit meta-package
projects. The committee is responsible for guiding, reviewing and finally
closing/approving the RFC.

# Template
Use the [Qiskit RFC template](0000-template.md) to prepare your RFC. Jupyter 
notebooks (.ipynb files) may also be used with the same template. 

## License
[License]: #license

This repository is licensed under

 Apache License, Version 2.0, ([LICENSE](LICENSE) or http://www.apache.org/licenses/LICENSE-2.0)
