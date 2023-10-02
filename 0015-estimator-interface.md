# Extended Estimator Interface

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0015                                         |
| **Authors**       | Ian Hincks (ian.hincks@ibm.com)              |
| **Deprecates**    | RFC that this RFC deprecates                 |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |


## Summary
The current `Estimator.run()` method requires that a user provide one circuit for every observable and set of parameter values that they wish to run.
(This was not always the case, but the history will not be discussed here.)
It is common, if not typical, for a user to want to estimate many observables corresponding to a single circuit. Likewise, it is common, if not typical, for a user to want to supply multiple parameter value sets for the same circuit.
This RFC proposes to fix these issues by changing the `Estimator.run()` in the following ways: 

 1. Take the transpose of the current signature; rather than accepting `circuits`, `parameter_values`, and `observables` as three different (iterable) arguments which are to be zipped together, instead localize the distinct tasks to be run via an iterable of triples `(circuit, parameter_values, observables)`.
 2. In combination with 1, extend `parameter_values` and `observables` to be array-valued, that is, add the ability to explicity specify multiple parameter value sets and observables for a single circuit.

## Motivation

Here is a summary of pain-points with the current `Estimator.run()` interface:

1. _Ergonomics._ It feels unnatural to invoke the estimator as `Estimator.run([circuit, circuit], observables=[obs1, obs2])` because of the redundancy of entering the circuit twice. 

1. _Trust._ The case `Estimator.run([circuit, circuit], observables=[obs1, obs2])` makes a user question whether the primitive implementation is going to do any optimizations to check if the same circuit appears twice, and if so, whether it will be done by memory location, circuit equality, etc.; the interface itself creates gap in trust.

1. _Clarity._ Without reading the documentation in detail, it's not obvious that the invocation `Estimator.run([circuit], observables=[obs1, obs2])` wouldn't cause `circuit` to be run with both supplied observables. In other words, that zipping _is always what's done to the `run()` arguments_ is common source of user confusion. Conversely, it's not clear that the way to collect all combinations of two observables and two parameter sets is to invoke `Estimator.run([circuit] * 4, observables=[obs1, obs2] * 2), parameter_values=[params1] * 2 + [params2] * 2`.

1. _Performance._ Given that `QuantumCircuit` hashing should be avoided and circuit equality checks can be expensive, we should move away from an interface which necessitates that implementations make associated tradeoffs in order to perform certain obvious optimizations. As one example, qubit-wise commuting observables like `"IZZ"` and `"XZI"` can both be estimated using the same simulation/execution, but only if the estimator understands that they share a base circuit. As a second example, when the circuits need to be seriazed before they are simulated/executed (e.g. runtime or pickling-for-multi-processing), it puts the onus on the primitive implementation to detect circuit duplications.
   

Here is why the detailed section of this proposal suggests "transposing" the signatureß and using array-based arguments with broadcasting:

 1. Transposing the signature will make it obvious what the primitive intends to do with each circuit.
 2. Transposing the signature will let us introduce and reason about the notion  "primitive unit of work", and carry it to other primitives.
 3. Array-based arguments will let users assign operational meaning to each axis (this axis is for twirling, that axis is for basis changes, etc.).
 4. Broadcasting rules will let users choose how to combine parameter value sets with observables in different ways, such as
    1. use one observable for all of N parameter value sets
    2. use one parameter value set for all of N observables
    3. zip N parameter value sets against N observables
    4. take all NxM combinations of N parameter value sets with M observables
    5. etc.

## User Benefit

All users of the primitives stand to benefit from this proposal.
Immediately, it will enable sophisticated and convenient workflows for power users though arrays and broadcasting.
However, standard broadcasting rules are such that the 0D and 1D cases will feel natural to existing users---0D is essentially what we already have, and 1D will be perceived as a simple-yet-welcome bonus.

For all users, the interface changes in this proposal will enable specific primitive implementations to enhance performance through reduced bandwidth on the runtime, compatibility with fast parameter binding, and multiplexing qubit-wise commuting observables.

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

Single task per run call.

## Questions
Open questions for discussion and an opening for feedback.

## Future Extensions
Consider what extensions might spawn from this RFC. Discuss the roadmap of
related projects and how these might interact. This section is also an opening
for discussions and a great place to dump ideas.

If you do not have any future extensions in mind, state that you cannot think
of anything. This section should not be left blank.

Primitive.run
Sampler.run
