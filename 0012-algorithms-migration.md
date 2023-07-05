# Plan to move `Qiskit/qiskit-terra/algorithms` module to `Qiskit-community/qiskit-algorithms` independent repo

| **Status**        | **Proposed**                                                                              |
|:------------------|:------------------------------------------------------------------------------------------|
| **RFC #**         | ####                                                                                      |
| **Authors**       | [Elena Peña Tapia](https://github.com/ElePT), [Steve Wood](https://github.com/woodsp-ibm) |
| **Submitted**     | 2023-07-05                                                                                |
| **Updated**       | YYYY-MM-DD                                                                                |


## Summary

This RFC was created to document the discussions around the future of the `qiskit.algorithms` module.
This module provides a series of interfaces and implementations of commonly used quantum algorithms based on 
Qiskit's core components. Historically, the module was created as part of Qiskit Aqua, together with the now
community-based application modules (Nature, Machine Learning, Optimization, Finance). It currently "lives"
in `qiskit-terra`, and over the past year has been refactored to use the Qiskit Primitives and sustain the 
Opflow and Quantum Instance deprecations.

The content of `qiskit.algorithms` is heterogeneous. Some components are algorithm implementations, while others
are "sub-routines" or algorithm utilities, such as gradients, optimizers or state fidelities. Some of these
utilities are implementatons from scratch, others are wrappers over commonly used libraries (such as the 
SciPy optimizers).

With Qiskit moving towards a leaner definition, we are now revisiting the future of this module. 
We would like to tell apart the fundamental components for algorithm design from algorithm implementations, and 
find a better suiting location for both of them while minimizing the disruption caused to users.


## Design Proposal

There are several design decisions to be made as part of this "`qiskit.algorithms` revision":

1. Which classes stay in `qiskit-terra`
2. Where do they stay within `qiskit-terra` (i.e. do they stay in `qiskit.algorithms`? or is the module 
renamed or split to better define its purpose?)
3. Where do the "non-chosen" classes go. We have to keep in mind that these algorithms are extensively used by
the community and until now, they have been one of the main access points to our stack. Even if we stop their 
maintenance, there would ideally still be a self-contained, installable package that the community can access
while they transition to the new alternatives. 

### Main Idea

We believe that fastest way to disentangle `qiskit.algorithms` from `qiskit-terra` while crating minimal 
disruption to users would 
be to create a standalone `qiskit-algorithms` repo with a similar setup to the applications repos (including CI, docs...),
perform a history-preserving copy of the content of `qiskit.algorithms` to `qiskit-algorithms`, and have a temporary
namespace redirect during the deprecation period that will raise a deprecation warning asking users to change the import.
This is the strategy that was followed when `qiskit-aer` was moved out of the providers. 

In the event that we decided to keep certain algorithms in `qiskit-terra`, there would be two possible paths:

**Path 1**

1. All contents of `qiskit.algorithms` are moved over to `qiskit-algorithms`, and the module as a whole
is deprecated.
2. Some components are moved back to `qiskit-terra`, potentially to a new location 
(`qiskit.gradients`, `qiskit.optimizers`, `qiskit.state_fidelities`...), and are potentially refactored during this move.

This alternative allows to have a "clean slate" for developers with the downside of potentially moving some components 
out of qiskit and back into qiskit again in a short period of time. The migration path is easier in this alternative, but 
deprecations are a huge hassle for users, and if we 
are sure of which components we want to keep, (if we want to keep any) this might not
be the best way (or the most popular) from a user-friendliness point of view.

**Path 2**

1. Only part of the contents of `qiskit.algorithms` are moved over to `qiskit-algorithms`, and classes
are deprecated individually.
2. This allows for refactorings to happen in-place as a second step

This alternative is more conservative, and requires a better understanding of what the future components of 
`qiskit.algorithms` will be, and whether we want to keep the module's name and current structure. The plans
to release Qiskit 1.0 should also be taken into account, as they imply a commitment to whatever is kept 
in Qiskit. 


After analizing pros and cons for both options, our current plan is to design a migration plan that
follows **Path 1**. Should new considerations be taken into account, this plan could change.

### Migration plan

**Repos immediately affected by the plan**: `Qiskit/qiskit-terra` and `Qiskit/qiskit-tutorials`

### qiskit-terra

1. Open a PR to delete `qiskit.algorithms` folder and add a namespace redirect, and a single (pending_)deprecation 
message telling users to change their imports to `qiskit_algorithms`. The redirect will allow existing code to 
continue with the message informing them of the change. 
2. Add a dependency of `qiskit_algorithms` to qiskit-terra so that installing terra will also install the 
algorithms until the deprecation period is over. [Blocker: the separate algorithms package would have to 
be ready and be published ahead of terra releasing]
3. What should we do with the algorithms migration guide? we could leave it in terra during the deprecation period, 
for users still coming to/using 
`qiskit.algorithms` but make a copy in `qiskit-algorithms` as it makes sense there too (maybe the copy needs some editing 
for references etc). After the deprecation period, we can remove it from terra.
4. What about already deprecated algorithms (i.e HHL)? Given that the deprecation period for these algorithms
will already be over
by the time we make the move, and links to the textbook tutorials have already been specified as an alternative
to these classes, the deprecated algorithms probably don't need to be moved over to the new location, and can 
be directly removed.
5. During the deprecation period, we can focus on moving issues, closing pending PRs, and finishing up other details

### qiskit-tutorials

1. There is already a plan to relocate the tutorials from `qiskit-tutorials` to `qiskit-terra`
https://github.com/Qiskit/qiskit-tutorials/issues/1473. Instead of relocating to terra, we propose to move the tutorials
to ``qiskit-algorithms/docs``` and be published as part of its docs as we do for Nature, ML et al.

### qiskit-algorithms

1. Create a new `qiskit-algorithms` repo and perform a history preserving copy plus a move, and whatever namespace 
fix-ups are needed for imports, deprecation messages of the old algos etc. We have done this history preserving 
copy in the past.
2. This new repo would need CI setup, the qiskit bot done to facilitate releases, the docs setup etc so they can 
publish out like apps repos do. And as mentioned above the docs can include a copy of the algorithms migration - 
maybe the short url points to this published copy.

### docs
1. Similarly to the application modules, the algorithm docs could be rendered in `qiskit.org/ecosystem/algorithms`
2. During the deprecation, a redirect could be set up to go from `qiskit.org/documentation/` to `qiskit.org/ecosystem/algorithms`


### Alternatives considered but disregarded

1. Splitting `qiskit.algorithms`, keeping some algorithms, moving others to different locations (such as the apps modules):
This idea was disregarded because of the disruption it will cause to users, to libraries for which `qiskit-algorithms` is
a dependency (i.e, apps modules), and the lack of consensus on which algorithms go where. 
2. Making `qiskit-algorithms` a git submodule of Terra instead of doing a redirect. The sub-module is linked by a 
specific commit as far as Steve recalls and its something that got discussed in the early days but was not well liked. 
The namespace strategy followed with qiskit.providers.aer redirecting to qiskit_aer works and keeps 
things more independent.


### Timeline

As this is an urgent move, the deprecations and new installable package should be in place for the terra 0.25 release. 
Docs can be updated in between releases.
The deprecation period should allow for the final decoupling to happen for 0.26.

## Questions

## Future Extensions
