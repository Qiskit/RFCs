# Run-time classical calculations within circuits

| **Status**        | **In development**                           |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com)          |
| **Submitted**     | 2022-07-15                                   |
| **Updated**       | 2022-07-15                                   |


## Summary

To support dynamic circuits, we need to be able to represent general classical
values and manipulation of these values within circuits.  Instructions and other
gates should be able to be parameterised in terms of these run-time values.


## Motivation

The OpenQASM 2-like model of circuit execution previously in use does not allow
any interaction with classical values during execution of a circuit, other than
conditioning a single gate on whether a register has one exact numeric value.
This is insufficient for describing quantum algorithms, which have non-trivial
classical processing mixed in with the quantum components.  In the near term, we
still expect to use QPUs as a co-processor in conjunction with a full classical
execution environment, but as control systems evolve, we want to allow users to
perform more classical operations within the description of a hybrid
quantum–classical programme.


## Top-level design

There are many steps to getting to a full, sensible representation of hybrid
quantum–classical programmes.  We have not designed all of these in detail right
from the start, as this is simply too large a task.  We will break down the
levels in more detail as we approach them.

The goals are:

- Add a representation of a "run-time classical value", which has an associated
  classical type.
- Allow operations to be passed these run-time values as parameters.
- Add operations that manipulate these run-time values (such as addition,
  less-than comparison, etc).
- Allow circuits to define run-time inputs and outputs.
- Add a representation of subroutines.

We need to progress towards these goals without breaking too much backwards
compatibility, which is tricky as these are changes to core data structures.
The following points are approximately in the order they should be implemented,
purely to get run-time parameters working.

- [x] Make the "instruction context" object in `QuantumCircuit` more extensible
  (Qiskit/qiskit-terra#8093).

  With run-time data, `QuantumCircuit` (and the transpiler's `DAGCircuit`
  intermediate representation) need to track more data dependencies between
  objects.  Adding a more formal "context" object for `QuantumCircuit` creates
  better separation of concerns between instruction definitions and information
  about their calls, and reduces memory usage.

  This step comes first because getting it right makes it easier to maintain
  backwards compatibility with all other steps; fixed-length, fixed-type tuples
  are inherently non-extensible, and so a more flexible data container is
  needed.
 
- [ ] Move all run-time operands of `Instruction` instances into the instruction
  contexts to be managed by `QuantumCircuit`.

- [ ] Add a new type to represent run-time classical values, including a
  representation of the type system for these objects.

- [ ] Add a representation of expressions on run-time classical values.

- [ ] Add circuit instructions that represent mutation of run-time values.

There are more goals that are part of expanding this run-time classical
computational support, but are not blocked on the above points, nor (in some
cases) on each other:

- [ ] Support the current `ControlFlowOp` classes throughout all transpiler
  passes packaged in Terra.  In many cases this just involves recursing the
  passes into the nested objects, but in others, such as routing what appear to
  be multi-qubit blocks that do not need full connectivity, it is non-trivial
  how to proceed.

- [ ] Introduce more control-flow-graph-like features to the transpiler's
  intermediate representation.  This is necessary both for better
  representations of what the execution model actually is for a given circuit,
  but also for efficiency in the transpiler.  The current embedding of
  `QuantumCicuit` within the `for`, `if` and `while` constructs means that these
  remain `QuantumCircuit`s when the transpiler representation is constructed,
  which leads to many inefficient conversions across all the passes.


## Terminology

There's a few words that are very similar, and we are partially through
repurposing some of our terminology from long-standing forms to new ones, which
can be confusing.  I will try to stick to the following:

- "operation": an object representing some run-time quantum operation (like an
  `rz` or a `measure`), that is made into an "instruction" when applied to a set
  of qubits, classical bits and (later) parameters.  In many/all of the
  `standard_gates` cases, the current `Gate` subclasses will be able to become
  these "operations" once this RFC implementation is complete, and will be
  stateless.

- "operands": the objects an "operation" is applied to.  At present, this is the
  qubits and clbits, but this RFC also describes the move to treat "run-time
  parameters" as operands as well.

- "context object": a container object that holds an "operation" and its
  "operands", and any additional application-specific knowledge such as
  scheduling information.  `QuantumCircuit.data` is a list of context objects,
  but it is not the only one; the nodes in `DAGCircuit` and `DAGDependency` also
  hold context of their applications.  The Python types representing these three
  types of context object do not need to be the same.

  Within `QuantumCircuit`, the context object is called `CircuitInstruction`.
  Within `DAGCircuit` is is currently `DAGOpNode`, and within `DAGDependency`
  it is `DAGDepNode`.

- "run-time parameters": parameters that _may_ be set dynamically at run-time in
  the OpenQASM 3 model.  For example, the angle of an `RZGate` is a run-time
  parameter (whether it's set to a constant value or not), but the matrix
  representation of a `UnitaryGate` is not.

  Note that "parameters" is a little ambiguous already: we have
  `Instruction.params` and `QuantumCircuit.parameters` which mean subtly
  different things.  `Instruction.params` is currently a sort of "state" for the
  instruction instance in many cases, while `QuantumCircuit.parameters` contains
  `Parameter` (subclass of `ParameterExpression`) instances that have not yet
  been bound.  `Parameter` represents a different computational model from a
  time before dynamic circuits - these were placeholder values to allow
  calculation and some degree of transpilation with Terra to occur only once,
  and then multiple circuits to be quickly constructed by assigning the
  placeholders various different values.  This is different to true run-time
  parameters that might change value _within_ a circuit run, or may be
  considered inputs to a circuit which the executing environment treats as
  runtime parameters.

- "instruction instance": specifically an instance of the class `Instruction`.
  These are currently specific, mutable instances of certain quantum operations,
  which also contain some state of how exactly they are used at their particular
  usage site (like a condition).


## Not Considered

We do not include a new transfer specification (to replace `Qobj`) in this
document.  In the immediate term, we serialise Terra object representations into
OpenQASM 3.0 payloads.  In the future, we may want to move to something else for
our provider APIs, but is not necessary right now.


## Future Extensions

Things that do not have a concrete place on any current roadmap, but are
expected to be required in the longer term:

- Add a builder interface (DSL) for making creating and manipulating classical
  expressions less painful.  The initial plan laid out in this document is for
  early users to have to write Terra's internal representations in order to use
  the classical features, but in the longer term, we should make this far more
  user-friendly by allowing them to write the expressions more naturally in
  a DSL embedded in Python, and "interpreting" this into our own internal
  representations.

- Applying operations whose qubit operands are indexed by run-time expressions.

- Some level of classical optimisation.  This may well not be the purview of
  Terra, but we will still need a design for how we construct our groupings of
  classical instructions in order to pass off the optimisations to another
  package (whether something lower in the provider stack, or 


## Detailed Design

### Context objects as the scalar type of `QuantumCircuit.data`

Before this work, the "context object" of an operation and its operands within
`QuantumCircuit` was a 3-tuple of `(Instruction, List[Qubit], List[Clbit])`.
This was mostly fine for non-dynamic circuits, where backends only ran on static
instructions with static parameters.  It is missing some components, though,
that would be more appropriate for the context object, even in the old model.
A condition on the instruction and any scheduling information were attached to
the `Instruction` instance, but would be more appropriate on the circuit (and
transpiler) context objects, since this is where they're needed for tracking.

The tuple format kept us from moving forwards, however; we cannot add more
objects to the context without breaking all existing uses, since the length of a
tuple is a core part of its type.  To solve this, we move to an encapsulated
`CircuitInstruction` class that uses attribute access rather than sequence
position for its elements.  This allows us to add more elements without breaking
existing code by extending the sequence.

#### New user interface

For complete details, see Qiskit/qiskit-terra#8093.

The initial version simply adds the data-like container class:

```python
class CircuitInstruction:
    operation: Instruction
    qubits: tuple[Qubit, ...]
    clbits: tuple[Clbit, ...]
```

The new interface for accessing the elements of circuit data is by named access,
so the common iteration pattern `for inst, qargs, cargs in qc` is replaced by

```python
for instruction in circuit:
    inst = instruction.operation
    qargs = instruction.qubits
    cargs = instruction.clbits
```

In practice, most real use-cases of these need to access only one element or so,
or need all three simply to rebuild a new tuple.  The rebuild operation can be
done with a helper function `CircuitInstruction.replace(**replacements)`.

This has some slight differences to the original 3-tuple.  The `qubits` and
`clbits` fields are now `tuple` rather than `list`.  This was to reflect that
they are truly immutable fields, and to allow these instructions to be copied
more cheaply; tuples of immutable objects can safely be referenced from several
locations, so there is no cost to a "copy".

The name `operation` for the `Instruction` attribute is anticipating other work
about a new, less restrictive interface for adding objects to `QuantumCircuit`
before transpiler passes.  We expect this to be called `Operation`.  The name
`CircuitInstruction` was chosen to avoid the naming conflict with `Instruction`,
while changing the terminology so an "instruction" is a set of an "operation"
and its "operands" (and associated context about this instruction).

All code in the Terra repository in the distributed Python package and in the
tests was updated to use the new attribute access patterns.  This minimised the
performance hit, will hopefully make the deprecation period less painful in the
future, and provides example code of how this new class should be used.


#### Backwards compatibility notes

We do not add deprecation warnings for the initial implementation per the
deprecation policy.  We do not add any new fields in the initial implementation,
but the new class now allows us the freedom to add more.

`CircuitInstruction` fulfils the `Sequence` interface so it can be used in a
duck-typed manner like a tuple.  This is particularly important for having the
standard construct

```python
for inst, qargs, cargs in qc:
    ...
```

continue to work as expected.  Notably, the `qargs` and `cargs` (second and
third) arguments of the original tuple were `list` not `tuple`, so
`CircuitInstruction` creates lists when iterating.  The exact `qubits` and
`clbits` fields not being mutable lists in the new class is not a backwards
compatibility problem because actually modifying these lists within
`QuantumCircuit.data` was not a supported operation in any way, and would likely
have invalidated the entire data structure.

When the class is extended, the legacy `Sequence` support must continue to
behave as if it is the 3-tuple.  If any new attributes are added into the
iterable form, the `for` loop above will break.  Old code cannot be using any
new elements (since it doesn't know they're in the context element), so it's not
a problem that it cannot access them.  The upgrade path is for new code to use
attribute access, once they can depend on Terra > 0.21.

We do not need to implement other magic methods that `tuple` implements, since
this is an internal type, and it does not make any sense to call things like
`__add__` on it.  We are not concerned with `__hash__` because the original
3-tuple contained lists, which are not hashable, and so these context objects
have never historically been hashable.

The `InstructionSet` class also needed to be updated.  This is returned by
`QuantumCircuit.append`, and at present is predominantly used to propagate
conditionals (`InstructionSet.c_if`) back onto 


#### Future work

The `Instruction.params` attribute is discussed elsewhere in this document.
Aside from this, there is more state currently in `Instruction` that would be
best as circuit context.  These components are:

- `condition`: this is perhaps that main object keeping _all_ `Instruction`
  instances stateful right now, hurting our memory usage.  The circuit should be
  responsible for handling control-flow, and will be with the new
  `ControlFlowOp` instructions (although how these should be represented within
  the circuit and the `DAGCircuit` is not 100% fixed yet).

- `duration` and `unit`: this is scheduling information that is similarly
  related to one specific application of an operation, not just the general
  description of what may be applied.

Moving these from `Instruction` to the circuit-context instruction would pave
the way for massively reducing Terra's memory usage in transpilation.  Almost
all standard operations, such as `Measure`, `XGate`, `RZGate`, and so on could
eventually become singleton instances.  Moving these will initially require
similar sorts of backwards-compatibility shims that are used in the swap of
`Instruction.params`, but simpler, since these less complex objects.


#### Rejected ideas

Using `NamedTuple` or in some other way deriving from `tuple` for the class was
rejected.  Deriving from `tuple` gives increased performance, but it does not
solve the underlying problem; we need to break away from using a sequence of
fixed length for the context object.


### Moving `Instruction.params` to the circuit context

Once "parameters" can be classically manipulated, they can no longer be
considered part of the state of an `Instruction` (operation); they are
"operands" of the operation.  Making this distinction is useful to get more
state out of `Instruction` anyway, but is vital for dynamic circuits, where the
circuit and transpiler must track the data dependencies between different
classical parameters, and their effects on the scheduling of the various
instructions.

The `Instruction.params` field is currently used for a lot more than simple
numeric scalar parameters.  For example, `UnitaryGate` stores a
$2^n \times 2^n$ matrix in its `params`, containing the matrix form of the gate
to be applied.  We must make a new distinction between what are "run-time
dynamic parameters" and what is "operation-specific state".  We only want to
move run-time dynamic parameters; circuits do not need to track any
operation-specific state, and they should not need to know what data may be
contained.  We do not want `QuantumCircuit` to be responsible for managing any
fixed state of its operations, only things that other operations in
`QuantumCircuit` might affect.

As a rule of thumb, parameters should be moved into being the circuit's
responsibility if they can reasonably be represented by an OpenQASM 3 type, and
if a pulse-level calibration _could_ be specified for a specific object with
parameters set at runtime.  This is satisfied for `RZGate`, for example, but not
`UnitaryGate`.  For many gates, this distinction is similar to whether a
`ParameterExpression` could be used in that slot, since due to previous
constraints in OpenQASM 2, Terra mostly only contains `Gate` instances with
angle-like parameters.

Note that the "circuit context" is not only the scalar type of
`QuantumCircuit.data`; the container classes `DAGCircuit` and `DAGDependency`
also have to maintain this context.  The current transpiler paths for accessing
`Instruction.params` involve looking in `DAGOpNode.op.params` (where
`DAGOpNode.op` is the `Instruction` instance), but a complete logical move of
`Instruction.params` requires the parameters to be accessible from `DAGOpNode`.
This is similar to how qubits and clbits are accessible from `DAGOpNode`.  The
initial unit of work does not involve adding additional wires between nodes in
the DAG to represent shared parameters, because this stage does not add any
mutable run-time parameters.

There are several methods of `Instruction` that must change once parameters have
moved.  Some will involve new user-facing interfaces, some will modify return
values.  In cases where the return values must change, we must consider our
backwards compatibility guarantees, and may need to supply new methods, simply
to avoid breaking backwards compatibility.

Including only methods affected by _this_ part of the RFC, the `Instruction`
methods that need to change in some form are:

- `_define` and `definition`: the definition must treat the parameters somehow.
  The current interface bakes the parameters in, so calling
  `Instruction.definition.to_instruction()` will return an object where the only
  parameters are the `Parameter` instances needed to fulfil the circuit, but not
  the numeric parameters.  This is inconsistent with the new model; 

- `validate_parameter`: this will no longer be necessary.  Type-checking of
  dynamic parameters should be centralised, and the responsibility of circuit
  objects.  Checking of "state" attributes of instructions should be handled by
  normal Python properties.

- `is_parameterized`: this refers specifically to there being unbound
  `Parameter` instances in the `params` field, and so will no longer be
  necessary, since this will no longer be a property of instruction instances.

- `assemble`: the `Qobj` transport format is not part of any future plans for
  interoperability, so this function will eventually no longer be needed.  In
  the near term, it can take an optional `parameters` argument and be handled
  similarly to `to_matrix` (see below).

- `reverse_ops`, `inverse` and `repeat`: these will need to change to use the
  decomposition methods that replace `definition`, but otherwise will remain
  largely the same.

- `qasm`: this is only used by the QASM 2 exporter, and with parameters and
  types specified, this should largely be removed.  The QASM 2 exporter should
  know to treat `measure` and `reset` specially without this being a method of
  those objects.  Only `Gate` subclasses can be exported to OpenQASM 2, and this
  should be a standardised export; `Instruction` gaining a "parameter
  specification" object will provide all the information needed to the exporter
  without a special method.


In addition to this, the `Gate`-specific methods that need to change in some
form are:

- `to_matrix` and `__array__`: these currently take zero arguments, but the
  matrix form is dependent on the input parameters.  We can never make
  `__array__` take extra arguments, so parameterised gates will no longer be
  able to inject constructors into `np.array`, but we can add a `parameters`
  field to `to_matrix` without breaking backwards compatibility (make it
  optional at first, and then deprecate not passing the argument).

- `power`: unlike `repeat`, this only operates on numeric parameters, because it
  constructs the matrix directly.  To support this in the near term, we likely
  need to add the `parameters` argument in the same manner as `to_matrix`.  In
  the further term, we may want to change this behaviour to match the OpenQASM 3
  `pow` gate-modifier semantics more closely.

- `control`: this (and by extension `.circuit.add_control.control`) needs its
  logic updating to handle the new form of parameters not being accessible from
  the `Instruction` instance, and to build the definition in line with the
  changes to how `Instruction` decomposition will be implemented (*i.e.* the
  replacement for `Instruction.definition`).  This should not be too difficult,
  as the actual logic need not change, just the locations and objects used to
  represent the parameters.


#### New user interface

Summary of changes:

- A `parameters` attribute will be added to `CircuitInstruction`, `DAGOpNode`
  and `DAGDepNode`.  Its contents will be the run-time settable dynamic
  parameters of the contained operation.

- `Instruction.params` will (eventually) be deprecated and removed.  Any
  instructions using this to represent instruction-specific state should likely
  expose their state as custom attributes.  Further requirements on instruction
  state will likely come from discussion about the `Operation` protocol.

- `Instruction.assemble`, `Gate.to_matrix` and `Gate.power` will accept a new
  `parameters` argument.  They should throw an exception if any of the
  parameters are not numeric constants.

- `Instruction.definition` likely cannot be changed in a backwards compatible
  manner.  Instead, we can add a new method, `Instruction.decompose` that takes
  zero arguments, and returns a circuit that takes the same parameter
  specification as `self` as inputs.

- `QuantumCircuit` needs to gain a method of specifying run-time inputs.  At
  first, it is best that this is kept private so we have freedom to change the
  interface later.  This will be discussed in much greater detail later in this
  section.

- `circuit_to_instruction` and `circuit_to_gate` should parametrise based on
  circuit inputs, rather than unbound `Parameter` instances.

The rest of this section is commentary and more detail on many decisions.

When iterating through the context objects in a circuit, the parameters will be
accessible directly from the context object, rather than a subcomponent of the
operation:

```python
for instruction in circuit:
    assert isinstance(instruction, CircuitInstruction)
    instruction.parameters        # new form
    instruction.operation.params  # old form
```

Note that the two objects here are not going to contain precisely the same
things; `CircuitInstruction.parameters` contains only those parameters that may
be dynamic, whereas `Instruction.params` contains in many cases additional
objects that are best described as the fixed state of that `Instruction`
instance.

`QuantumCircuit.assign_parameters` will largely behave in the same way as it
does now; `ParameterExpression` is orthogonal to run-time parameters.  These
will be discussed in more detail later.  The "parameters" in `assign_parameters`
refers to the `Parameter` class, not this new description of run-time
parameters.



#### Backwards compatibility concerns

`Instruction.params` is a list, and several parts of Terra assign to the list,
including `QuantumCircuit.assign_parameters`.  For backwards compatibility in
dependencies, we need to maintain this behaviour: a user must be able to assign
to an `Instruction` instance's `params` field and mutate the contained list, and
have those changes propagated to the circuit.  This must work both before and
after the instruction is added to a circuit.  To allow both old and new paths to
work, parametrised instructions must be able to accept parameters at
initialisation (old style) as `RZGate(math.pi)`, and to be constructed without
parameters `RZGate()`.  In the old style, the contained parameters must
propagate through to the circuit context when the instruction is added to it.

We need both of these to work and be equivalent:
```python
# Old style
gate = RZGate(1.23)
circuit.append(gate, [0], [])

# New style
gate = RZGate()
circuit.append(gate, [0], [], [1.23])
```

Furthermore, after either of these lines, modifications to the parameters
through both `gate.params` and `circuit.data[0].parameters` should both work,
and both should reflect each other.

This implies that we cannot add the same parameterised instruction instance to a
circuit more than once, while backwards compatibility with the old interface is
still required.  This does not need to be a deep copy for these purposes; it's
enough to just have a different binding within the `params` field.

For this, we need context objects (`CircuitInstruction`, `DAGOpNode` and
`DAGDepNode`) to 

We already deep copy some parameterised instruction instances for a different
reason: if it contains `ParameterExpression`, we deep copy so modifications to
`Instruction.definition` during `QuantumCircuit.assign_parameters` will be
correctly handled by `ParameterTable`.  The new copying requirements expand this
slightly to all parameterised instructions, but this only needs to be a shallow
copy.  For performance, we can gate the copy behind whether the `Instruction` is
already present in a circuit; a reference to the containing context object must
be held by the `Instruction`, so it is a simple null-pointer check to determine
if an `Instruction` is bound elsewhere already.


#### Rejected implementations

For the back-referencing `parameter` shims, it would be simpler if the current
`Instruction.params` stayed the same, and the context objects gained special
look-up methods.  For example, `CircuitInstruction.parameters` _could_ look
in the contained `CircuitInstruction.operation` attribute instead.

This would be much simpler to implement, but it means that the preferred new
access patterns are what takes the performance hit from increased look-up times,
and does not allow us to realise any of the benefits of moving the parameters
until later.  The aim is that only users who are still using the backwards
compatibility shims should have to pay the costs for their existence, not
everyone else.
