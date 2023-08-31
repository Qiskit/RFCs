# Plan to move `Qiskit/qiskit-terra/algorithms` module to `Qiskit-community/qiskit-algorithms` independent repo

| **Status**        | **Proposed**                                                                              |
|:------------------|:------------------------------------------------------------------------------------------|
| **RFC #**         | 0013                                                                                      |
| **Authors**       | [Elena Peña Tapia](https://github.com/ElePT), [Steve Wood](https://github.com/woodsp-ibm) |
| **Submitted**     | 2023-07-05                                                                                |
| **Updated**       | 2023-08-31                                                                                |


## Summary

This RFC was created to document discussions concerning the future of the `qiskit.algorithms` module.
This module provides a series of interfaces and implementations of commonly used quantum algorithms based on 
Qiskit's core components. Historically, the module was created as part of Qiskit Aqua, together with the now
community-based application modules (Nature, Machine Learning, Optimization, Finance). 
It [currently "lives" in `qiskit-terra`](https://github.com/Qiskit/qiskit),  
and over the past year has been refactored to use the Qiskit Primitives and sustain the 
Opflow and Quantum Instance deprecations.

The content of `qiskit.algorithms` is heterogeneous. Some components are algorithm implementations, while others
are "sub-routines" or algorithm utilities, such as gradients, optimizers or state fidelities. Some of these
utilities are implementatons from scratch, others are wrappers over commonly used libraries (such as the 
SciPy optimizers).

With Qiskit moving towards a leaner definition and focusing on primitives as quantum hardware access model, 
the future of `qiskit.algorithms` is being revisited.
The classes in the module tend to hide the primitives interfaces and serve as convenience wrappers 
for abstractions, beyond the current definition of Qiskit.
As such, they do provide value to users to it would be important to 
find a better suiting location for them while minimizing the disruption caused to users.

**Note**: In this document, the terms `qiskit-terra` and `qiskit` are interchangeable.

## Design Proposal

There are several design decisions to be made as part of this "`qiskit.algorithms` revision":

1. Which classes stay in `qiskit-terra`
2. Where do they stay within `qiskit-terra` (i.e. do they stay in `qiskit.algorithms`? or is the module 
renamed or split to better define its purpose?)
3. Where do the "non-chosen" classes go

We have to keep in mind that these algorithms are extensively used by
the community and until now, they have been one of the main access points to our stack. For example, the 
`MinimumEigensolver` class is a fundamental 
dependency for Qiskit Nature, which relies on this definition to interface with classical chemistry codes via 
plugins. 

Even if we stop their maintenance, it would still be highly valuable to have a self-contained, installable package 
that the community can access and add as a dependency to avoid having to reimplement research code or libraries.

### Main Idea

We believe that fastest way to disentangle `qiskit.algorithms` from `qiskit-terra` while causing minimal 
disruption to users would 
be to create a standalone `qiskit-algorithms` repo with a similar setup to the applications repos (including CI, docs...),
perform a history-preserving copy of the content of `qiskit.algorithms` to `qiskit_algorithms`, and document
the migration path by raising a deprecation warning asking users to change the import.

This repository can be created regardless of whether we decide to keep some algorithms in `qiskit-terra` or not.
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

**Decision**

After analizing pros and cons for both options, our current plan is to design a migration plan that
follows **Path 1**. This allows us to focus on the move to the new repository as an initial step, without
deciding specifically which algorithms/utilities might be added back to Qiskit.
Should new considerations be taken into account, this plan could change.

### Migration plan

**Repos immediately affected by the plan**: `Qiskit/qiskit-terra` and `Qiskit/qiskit-tutorials`

### qiskit-terra

1. Open a PR to add a single deprecation message telling users to change their imports 
from `qiskit.algorithms` to `qiskit_algorithms`. The code will continue existing 
in `qiskit-terra` until the deprecation period is over [Blocker: release of `qiskit_algorithms-0.1.0`,
see below].
2. What should we do with the algorithms migration guide? we could leave it in terra during the deprecation period, 
for users still using  `qiskit.algorithms` but make a copy in `qiskit-algorithms` (maybe the copy needs some editing 
for references etc). After the deprecation period, we can remove it from terra.
3. What about algorithms that depend on already deprecated code (i.e opflow)? 
Given that the deprecated code will still be available in `qiskit.algorithms`, we can just migrate the non-deprecated
code to the new location and warn users about updating their code before migrating to the new package 
(direct to migration guides). This will provide a "clean slate" for `qiskit_algorithms`.
4. During the deprecation period, we can focus on moving issues, closing pending PRs, and finishing up other details

### qiskit-tutorials

1. There is already a plan to relocate the tutorials from `qiskit-tutorials` to `qiskit-terra`
https://github.com/Qiskit/qiskit-tutorials/issues/1473. Instead of relocating to terra, we propose to move the tutorials
to `qiskit-algorithms/docs` and be published as part of its docs as we do for Nature, ML et al.
2. During the deprecation period, a redirect could be set up to go from `qiskit.org/documentation/tutorials/algorithms/*` to 
`qiskit.org/ecosystem/algorithms/tutorials/index.html`
3. Which tutorials should be moved? All tutorials that use `qiskit_applications`. Other tutorials that build algorithms
from scratch (such as https://github.com/Qiskit/qiskit-tutorials/blob/master/tutorials/algorithms/09_IQPE.ipynb) are 
better suited as educational material.


### qiskit-algorithms

1. Create a new `qiskit-algorithms` repo and perform a history preserving copy plus a move, and whatever namespace 
fix-ups are needed for imports, deprecation messages of the old algos etc. We have done this history preserving 
copy in the past.
2. Perform a history preserving copy of the relevant tutorials in `qiskit-tutorials` plus necessary code updates 
(imports, etc)
3. This new repo would need CI setup, the docs setup, etc, so they can 
publish out like apps repos do. And as mentioned above, the docs can include a copy of the algorithms migration - 
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

## Open Questions

There are 2 main open questions left:

1. Are any algorithms going to stay in terra?
2. If so, where would they stay?

Under the assumption that we only want to keep 
"fundamental building blocks" for algorithm design, our current proposal would be to maintain
sub-routines and utilities (gradients, fidelities, optimizers that are not wrappers), 
and replace specific algorithm implementations with well crafted tutorials/documentation. Because of the independent
`qiskit-algorithms` repository, users that relied on other algorithm implementations would still have access to these 
classes.

Specifically, if we look at the current non-deprecated contents of the module, the proposed distribution would be:

- `amplitude_estimators` -> tutorial
- `eigensolvers` -> tutorial
- `gradients` -> **utility that stays**
- `minimum_eigensolvers` -> tutorial
- `optimizers` -> **those that don't wrap scipy ara a utility that stays**
- `phase_estimators` -> tutorial
- `state_fidelities` -> **utility that stays**
- `time_evolvers` -> tutorial
