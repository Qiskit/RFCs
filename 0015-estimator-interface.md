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
 2. In combination with 1, extend `parameter_values` and `observables` to be array-valued, that is, add the ability to explicity and conveniently specify multiple parameter value sets and observables for a single circuit.

## Motivation

Here is a summary of pain-points with the current `Estimator.run()` interface:

1. _Ergonomics._ It feels unnatural to invoke the estimator as `Estimator.run([circuit, circuit], observables=[obs1, obs2])` because of the redundancy of entering the circuit twice. 

1. _Trust._ The case `Estimator.run([circuit, circuit], observables=[obs1, obs2])` makes a user question whether the primitive implementation is going to do any optimizations to check if the same circuit appears twice, and if so, whether it will be done by memory location, circuit equality, etc.; the interface itself creates gap in trust.

1. _Clarity._ Without reading the documentation in detail, it's not obvious that the invocation `Estimator.run([circuit], observables=[obs1, obs2])` wouldn't cause `circuit` to be run with both supplied observables. In other words, that zipping _is always what's done to the `run()` arguments_ is common source of user confusion. Conversely, it's not clear that the way to collect all combinations of two observables and two parameter sets is to invoke `Estimator.run([circuit] * 4, observables=[obs1, obs2] * 2), parameter_values=[params1] * 2 + [params2] * 2`.

1. _Performance._ Given that `QuantumCircuit` hashing should be avoided and circuit equality checks can be expensive, we should move away from an interface that necessitates performant primitive implementations to go down this path. As one example, qubit-wise commuting observables like `"IZZ"` and `"XZI"` can both be estimated using the same simulation/execution, but only if the estimator understands that they share a base circuit. As a second example, when the circuits need to be seriazed before they are simulated/executed (e.g. runtime or pickling-for-multi-processing), it puts the onus on the primitive implementation to detect circuit duplications.
   

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

### Tasks

In this proposal, we introduce the concept of a Task, which we define as a single circuit along with auxiliary data required to execute the circuit relative to the primitive in question. This concept is general enough that it can be used for all primitive types, current and future, where we stress that what the “auxiliary data” is can vary between primitive types. 

For example, a circuit with unbound parameters (or in OQ3 terms, a circuit with inputs) alone could never qualify as a Task for any primitive because there is not enough information to execute it, namely, numeric parameter binding values. On the other hand, conceptually, a circuit with no unbound parameters (i.e. an OQ3 circuit with no inputs) alone could form a Task for a hypothetical primitive that just runs circuits and returns counts. This suggests a natural base for all Tasks:

```python
BaseTask = NamedTuple[circuit: QuantumCircuit]
```

For the `Estimator` primitive, in order to satisfy the definition as stated above, we propose the task structure

```python
ObservablesTask = NamedTuple[
    circuit: QuantumCircuit, 
    parameter_values: BindingsArray, 
    observables: ObservablesArray
]
```

We expect the formal primitive API and primitive implementations to have a strong sense of Tasks, but we will not demand that users construct them manually in Python as they are little more than named tuples, and we do not wish to overburden them with types. This is discussed further in the “Type Coersion” section.

## Detailed Design
Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

## Alternative Approaches

An alternative is to consider letting the `run()` method accept, effectively, only a single `ObservablesTask`:

```python
Estimator.run(cirucuit, parameter_values_array, observables_array)
```

This has the advantage of a simpler interface, where multiple tasks could be run 
by invoking the estimator multiple times. The disadvantages, which we feel are significant enough to forego this simplification, are that:

 1. For real backends, the user would lose the ability to cause multiple types of circuits to be loaded into the control hardware at one time. For example, if using an estimator to perform randomized benchmarking, each circuit depth would need to be a separate job.
 2. It would be difficult for implementations that include mitigation to share resources between tasks. For example, if different tasks represent different trotter step counts, there would need to be a complicated mechanism to share learning resources---that are specific to the application circuits---between multiple jobs.

## Questions
Open questions for discussion and an opening for feedback.

## Future Extensions

In this proposal we have typed circuits as `QuantumCircuit`. It would be possible to extend this to a `CircuitLike` class which could be as simple as `Union[QuantumCircuit, str]` to explicitly allow OpenQASM3 circuits as first-class inputs.