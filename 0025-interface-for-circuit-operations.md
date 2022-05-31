# `Operation`: the interface for valid `QuantumCircuit` operations

| **Status**        | **Proposed**                                 |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com)          |
| **Deprecates**    | None                                         |
| **Submitted**     | 2022-05-31                                   |
| **Updated**       | 2022-05-31                                   |


## Summary

Currently, only subclasses of `Instruction` can be put on `QuantumCircuit`, but
this interface has become unwieldy and includes too many methods and attributes
for general-purpose objects.  We propose to put a new interface in place that is
far more stripped down, and to remove transpilation-specific methods from the
objects themselves, and move them to make them transpiler behaviour.  We propose
that this new interface will be called `Operation`.


## Motivation

- `Instruction` as a complete type has too many unrelated attributes and methods
  from the base interface, which not all "quantum operations" can reasonably
  fulfil.  For example, `definition` (which is already circumvented in several
  cases), `duration` and `unit` (which only have meaning after transpilation),
  and `assemble` (which supposes that the object is immediately executable).

  Some of these are fine _after transpilation_, but are too restrictive for
  `Instruction` to be the only input type for the transpiler.

- `Instruction` is defined as part of `qiskit.circuit`, which poses cyclic
  dependency issues for some lower-level objects (e.g. those based in
  `quantum_info` types).

- Having a simpler, abstract interface (`Operation`) allows us to break the
  cyclical dependencies (it can be the lowest part of Terra), make more objects
  suitable for being "transpiler input", and reduce the memory/implementation
  complexity for those objects that can be.

- `Instruction` need not be replaced by this; we still want something to
  represent "a transpiled operation".  This is a greater restriction than "an
  operation that can be transpiled", so `Instruction` can (and will) also
  implement `Operation`.

- Defining an immutable interface lets us re-use the same objects more during
  transpilation, which will save a lot of memory.  For example, we will not need
  to repeatedly copy `definition` circuits just in case the parameters are
  mutated by some later pass, and we will not need to copy in case the
  `condition` of the instruction is modified.


## User Benefit

- Users will have more freedom to work with a variety of objects (such as
  `quantum_info.Clifford`) and store them directly onto circuits, letting them
  interact again with the rich objects.

- Developers gain a backwards-compatible way to reduce the dependence on some of
  the memory-hungriest components of `Instruction`.

- Transpiler pass writers can use the rich methods of the objects without the
  additional memory usage (and type-unsafety) of wrapping each thing it its own
  `Instruction` subclass.

- Transpiler pass writes will be able to more easily write synthesis methods
  that use more information from the transpiler.

- Researchers will be able to more easily "plug in" their own synthesis methods
  for higher-order objects at transpilation run time, without needing to modify
  the objects themselves, or mess with pass managers.


## Design Proposal

We propose an abstract interface that defines a circuit operation that can be
input to the transpiler.  We call this `Operation`.

The propsed attributes on `Operation` are:

- `name`: the name identifier, to provide a link between objects and `Target`,
  the API that backends use to describe what operations they support.

- `num_qubits`: the logical number of qubits this instruction operates on.
  Synthesis methods may use more qubits than this to implement the operation,
  but this is the number of "data" qubits.  This is to enable quantum resource
  tracking and type safety.

- `parameter_spec`: number and types of the parameters that need to be supplied
  to this object at run-time (notably _not_ the parameters themselves).  This is
  to enable classical resource tracking and type safety.

- `state`: an opaque object that contains the full state of the object, which
  can be passed to some as-yet-undefined method to reconstruct the object.  This
  is for serialisation.

Some things `Operation` should not do:

- force objects to make decisions about their synthesis
- force all objects to be stateful (i.e. there should be a valid implementation
  of `HGate` that is a singleton)
- require objects to construct / be able to construct a `QuantumCircuit` through
  any of the abstract methods

These things mean that the following properties or methods should _not_ be part
of `Operation`:

- `condition`: this should be part of separate classical control flow, and its
  stateful nature is one of the current reasons for a lot of copying of
  `Instruction` instances.

- `duration` and `unit`: these are properties of the output of the transpiler,
  and should also depend on the particular qubits in use.  This likely wants to
  become part of `CircuitInstruction` (see Qiskit/qiskit-terra#8093).

- `assemble`: this is particular to the conversion to the `Qobj` type, which may
  not exist in the long-term, but even in the short-term, this is conversion
  behaviour and the interfaces should be well-enough defined such that this does
  not need to be specialised.

- `definition` or `_define`: this ties synthesis to the object, causing us to
  carry around extra circuits (and frequently copy them).


## Alternative Approaches

It is in theory possible to maintain the status quo, and have `Instruction` be
the definition of the transpiler-input interface.  Some things (such as
pluggable, controllable synthesis passes) are still possible to do in this
structure.  The memory problems are (at best) exacerbated by `Instruction`,
though, and it is difficult to fix this will maintaining backwards
compatibility.  This also would do nothing to address the general design
problems with operations currently needing to define their own synthesis.

The memory usage is already showing problematic strain for scalability.  See:

- Qiskit/qiskit-terra#7485
- Qiskit/qiskit-terra#6991
- Qiskit/qiskit-terra#5895

Having all operations have to define their own `definition` causes cyclic
dependency and hierarchy issues: `quantum_info` classes generally should not
need to care about the circuit model, since they are using a different
formalism.  This also means that objects with more than one synthesis method
have to define a "preferred" synthesis method.  This is an issue for things like
the multi-controlled X.


## Questions

1. At this point, the exact requirements for `Operation` are not fully
   finalised.  This RFC will not be ready for implementation until they are.

2. Will classical operations also be `Operation`, or will we do something
   different with them?

3. What information about necessary qubits should we expose, bearing in mind
   that the "number of qubits needed" may be dependent on the synthesis method?


## Future Extensions

This has generally come up as part of two additional strands of work:

- richer, more controllable synthesis on complex objects
- separation of concerns between "operations" and "operands" in `QuantumCircuit`

Defining `Operation` completely ties a lot into both of these.

For the richer synthesis to be a powerful, (relatively) easy-to-use tool, we
need to make sure that the transpiler has all the information it needs within
the `Operation` class to do its job while treating the objects as otherwise
entirely opaque.  The more the transpiler needs to know about an object, the
more onerous and memory-hungry it is for general-purpose objects to implement
the interface, which is a problem for scaling.

The separation of concerns between the operations, operands, and circuit-level
properties of given instructions is inherently tied in, because some components
of `Instruction` can be made "operands", rather than needing to be tied to the
instance.  This is immediately `params` as part of `Instruction`.
