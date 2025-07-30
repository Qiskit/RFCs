# Low level sampler

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0023                                         |
| **Authors**       | Salva de la Puente (salva@ibm.com),  Antonio Corcole, Ian Hincks    |
| **Deprecates**    | TBD                 |
| **Submitted**     | 2025-07-30                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary

In Qiskit, the quantum computer interface is built around primitives, which are
the fundamental operations executable on a quantum circuit. Currently, Qiskit offers
two such primitives: the Sampler and the Estimator. The Estimator leverages advanced
error mitigation techniques to enhance the quality of results from noisy quantum
computers. On a noiseless quantum computer, the Estimator could instead be constructed
atop the Sampler, simplifying its role. However, given current noise levels, robust
error mitigation is essential, requiring extensive profiling through thousands or even
millions of circuit variations. Transmitting these variations across cloud environments
is inefficient, but this can be optimized using a "samplex" a domain-specific language
(DSL) program describing the way of producint these variations. The samplex enables
efficient server-side generation of variations, with results transmitted back as samples
accompanied by metadata, facilitating client-side implementation of error-mitigated
estimations.

## Motivation
- Why are we doing this?

We are pursuing this development to address the growing demand within the quantum information science community for advanced error mitigation techniques. Users and researchers seek finer control over these techniques to improve the reliability of quantum computations, especially on noisy quantum hardware. Vendors like IBM have identified that users desire capabilities for precise noise learning, twirling, and expectation value calculations. By optimizing the process with tools like the "samplex" DSL, we can enhance efficiency in generating circuit variations server-side, reduce data transmission overhead, and support robust client-side error-mitigated estimations. This approach ultimately aims to deliver higher-quality quantum computing results while meeting the sophisticated needs of the community.

- What will this enable?
- What will be the outcome?
- Who will benefit?

## User Benefit
- Who are the target users of this work?
- How will users or contributors benefit from the work proposed?

## Design Proposal
This is the focus of the document. Explain the proposal from the perspective of
educating another user on the proposed features.

This generally means:
- Introducing new concepts and nomenclature
- Using examples to introduce new features
- Implementation and Migration path with associated concerns
- Communication of features and changes to users

Focus on giving an overview of impact of the proposed changes to the target
audience.

Factors to consider:
- Performance
- Dependencies
- Maintenance
- Compatibility

## Detailed Design
Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

## Alternative Approaches
Discuss other approaches to solving this problem and why these were not
selected.

## Questions
Open questions for discussion and an opening for feedback.

## Future Extensions
Consider what extensions might spawn from this RFC. Discuss the roadmap of
related projects and how these might interact. This section is also an opening
for discussions and a great place to dump ideas.

If you do not have any future extensions in mind, state that you cannot think
of anything. This section should not be left blank.
