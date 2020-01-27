# Third-party package contribution guideline and monitoring

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Luciano Bello (@1ucian0), Toshinari Itoko (@itoko) |
| **Deprecates**    |                  |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary
There are many contributions that could be merged into Qiskit eventually but they have not yet examined well in terms of usability, performance or code quality.
Reviewing direct PRs of such contributions is usually very hard, takes too much time to make it mergeable, and might frustrate both of contributors and reviewers.
This RFC proposes to allow a relatively casual way of contributions, i.e. creation of 3rd party package with Qiskit by providing an official guideline for such packaging.
It will encourage broader contributions to Qiskit, and promote the growth of Qiskit community. 


## Motivation
- Why are we doing this?
There are many contributions that could be eventually merged into Qiskit but they have not yet examined well in terms of usability, performance or code quality. For examples, 
1. Apps/features created in community events (e.g. [Qiskit Camp](https://qiskit.org/events/camp-2020/))
2. Modules proposed without RFC:
[#2867](https://github.com/Qiskit/qiskit-terra/pull/2867.
3. Transpiler passes developed for writing papers:
[#1803](https://github.com/Qiskit/qiskit-terra/pull/1803),
[#2860](https://github.com/Qiskit/qiskit-terra/pull/2860).

In fact, some of direct PRs of those contributions are undergoing substantial review over several months or more.

- What will this enable?


- What will be the outcome?


- Who will benefit?

## User Benefit
- Who are the target users of this work?
- How will users or contributors benefit from the work proposed?

## Design Proposal
-provide a guideline describing how to package their own module (with qiskit as a requirement).

## Detailed Design
Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

## Alternative Approaches
- Creating a new repo for collecting 3rd-party contributions
The original author has full control on their code and is in charge of packaging, testing, licensing and so on.. (If Qiskit owners had to do those things, this proposal could not make the contribution process faster.) 
So the 3rd-party contributions should be maintained in their own repos i.e. 3rd-party repos.

## Questions
-build monitoring (or benchmarking) system
-create a workflow to add packages in that benchmark

## Future Extensions
Consider what extensions might spawn from this RFC. Discuss the roadmap of
related projects and how these might interact. This section is also an opening
for discussions and a great place to dump ideas.

If you do not have any future extensions in mind, state that you cannot think
of anything. This section should not be left blank.
