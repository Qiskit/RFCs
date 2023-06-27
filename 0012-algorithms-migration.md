# Plan to move `Qiskit/qiskit-terra/algorithms` module to `Qiskit-community/qiskit-algorithms` independent repo

| **Status**        | **Proposed**                                                                              |
|:------------------|:------------------------------------------------------------------------------------------|
| **RFC #**         | ####                                                                                      |
| **Authors**       | [Elena Peña Tapia](https://github.com/ElePT), [Steve Wood](https://github.com/woodsp-ibm) |
| **Submitted**     | 2023-06-27                                                                                |
| **Updated**       | YYYY-MM-DD                                                                                |


## Summary

As part of Qiskit's move towards a leaner definition, `qiskit.algorithms` should no longer be part of 
`Qiskit/qiskit-terra`, soon to be `Qiskit/qiskit`, and instead become an independent repository in the Qiskit
Ecosystem: `Qiskit-community/qiskit-algorithms`

This document traces a plan for this migration, to inform the community about the potential for upcoming breaking
changes, and to ask help identifying and defining mitigations for usages that may be impacted by this change.


## Design Proposal - Migration Plan

### Repos affected by the change
`Qiskit/qiskit-terra` and `Qiskit/qiskit-tutorials`

### Migration plan

We believe that fastest way to disentangle `qiskit.algorithms` from `qiskit-terra` while crating minimal disruption to users would 
be to create a standalone `qiskit-algorihtms` repo with a similar setup to the applications repos (including CI, docs...),
perform a history-preserving copy of the content of `qiskit.algorithms` to `qiskit-algorithms`, and have a temporary
namespace redirect during the deprecation period that will raise a deprecation warning asking users to change the import.
This is the strategy that was followed when `qiskit-aer` was moved out of the providers. 

The step-by-step breakdown of the plan is as follows:

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
4. During the deprecation period, we can focus on moving issues, closing pending PRs, and finishing up other details

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


### Alternatives considered but disregarded

1. Splitting `qiskit.algorithms`, keeping some algorithms, moving others to different locations (such as the apps modules):
This idea was disregarded because of the disruption it will cause to users, to libraries for which `qiskit-algorithms` is
a dependency (i.e, apps modules), and the lack of consensus on which algorithms go where. 
2. Making `qiskit-algorithms` a git submodule of Terra instead of doing a redirect. The sub-module is linked by a 
specific commit as far as Steve recalls and its something that got discussed in the early days but was not well liked. 
The namespace strategy followed with qiskit.providers.aer redirecting to qiskit_aer works and keeps 
things more independent.


### Timeline

As this is an urgent move, the deprecations and new installable package should be in place for the terra 0.25 release, 
and the final move should happen on 0.26.

## Questions

## Future Extensions
