# Qiskit middle-end IR for fault-tolerant compilation

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | #### |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com), Kit Barton (kbarton@ca.ibm.com) |
| **Submitted**     | 2026-09-08 |


## Summary

Qiskit will introduce a new middle-end intermediate representation (MIR) for fault-tolerant compilation.
This will be one section of full-path compilation from high-level program description down to specific hardware.

It will be the primary target for higher-level applications representations to lower to, both from inside and outside Qiskit.
The IR will support hardware-informed optimisations and rewrites, including incremental physical-qubit/module assignments, but not be a concrete representation of any hardware/error-code family.
The IR will be subsequently lowered to a variety of code/hardware-specific representations, depending on lower-level backend requirements.

The quantum instruction set will permit several different computational models, and expect pipelines to specific backends to narrow the used set down before lowering.
It will include Pauli-based computation, Clifford tableaus and explicit low-arity gates.

Some examples of transformations we expect to able to perform on this IR:

- placement of "virtual" qubits onto hardware qubits and modules
- operation translation
- explicit Clifford-correction tracking
- continuous-rotation synthesis to discrete gates (e.g. `rz` synthesis)

The IR will permit a small amount of classical computation and control flow, but this will deliberately limited during the first designs.

The IR will simultaneously model both "virtual" and "physical" qubit addressing.
Virtual addressing uses qubits, groups and connectivity defined inline in the IR, prior to allocation to specific physical qubits and groups.
The properties of each qubit and group in the physical-addressing mode will be communicated by an abstract interface implemented by backends.


## Setting and scope

### Motivation

Fault-tolerant compilation will have a different shape to NISQ-era compilation.
Qiskit's existing representations are designed and optimised for gate-model computation.
Trying to evolve `QuantumCircuit`/`DAGCircuit` in place risks comprising their suitability on current devices, and the split focus will stymie development of new forms.

Neither the ideal input forms nor the exact quantum-computer ISAs are precisely known yet, but we must begin trying to graduate individual research prototypes of components into research pipelines.
The clearest required component right now is a middle-end representation that can be targeted by various high-level algorithm-implementation languages, then lowered to a variety of implementation-specific backends (e.g. a compiler to one particular bicycle ISA).

#### Detail on deficiencies in `QuantumCircuit`/`DAGCircuit`

- There is no concept of "qubit grouping".
  We expect many FT backends to have instructions that "address" qubits in groups.
  `DAGCircuit` cannot reason about this; it assumes operations on disjoint qubits are parallelisable.

- `DAGCircuit` has poor abstractions for partial hardware assignments.
  It is hard to reason about qubit allocation and de-allocation, other than globally.

- The classical-value representations are very poor.
  `Clbit` semantics are inherited from `Qubit`, but classical bits are not generally linear types.
  Measurement feed-forward has unnecessary restrictions on re-ordering in `DAGCircuit`,.

- The representations of Pauli-based compilation are inefficient.
  There is some scope for in-place improvement, but it's still bounded by historical baggage.

- The representations of multi-qubit instructions are inefficient.
  `QuantumCircuit`/`DAGCircuit` are very memory efficient in representing gates on small numbers of qubits, where the combinations are small.
  These benefits do not scale well to many-qubit instructions, such as those found in PBC.

- `DAGCircuit` is a _pure_ data-flow representation defined only by qubit/clbit/var objects.
  There is no (common/easy) way to choose a specific linearisation during lowering, and the costs of maintain the data-flow analyasis must be paid by every mutation.
  Backend compilers have been known to be sensitive to the precise linearisation.


### User benefit

There are several user personas in mind, which have significant overlap in the general field of "compiler research" and "end user".
Here we pull out a couple of the more separable parts:

- *High-level algorithms language designers*.
  Will have a centralised IR to target, without being _required_ to know about specifics of the QEC backend structure.
  We anticipate that any Qiskit MIR program will be _possible_ to lower to any target backend, though we will provide abstractions in the IR (such as a generic concept of qubit "groups") that can be used to specialise lowerings on a per-pipeline basis.

- *Backend ISA developers*.
  Qiskit will provide a centralised abstraction / quantum virtual-machine representation for middle-end optimisation.
  We intend that large amounts of the compilation _can_ be done at the Qiskit MIR level, for any hypothetical backend (and Qiskit MIR will expand as we identify more places for suitable abstraction), and the ISA-specific lowering will be relatively cheap.
  This allows Qiskit to shoulder the greater burden of IR specification, compiler infrastructure, code optimisation, etc, which in turn allows the same effort to be shared among more users.

- *Quantum-computer users*.
  Most obviously: we need a platform for developing fault-tolerant compilers so users can access the hardware.
  These users benefit by us accelerating the work to get them access to the hardware.


### Design principles

Some general principles that guide decisions throughout:

- The initial design will not be complete and must permit extension without breakage.

- The API boundaries at the upper and lower ends are both movable.
  These are still active fields of research; we should support on-the-fly custom operations, and consider moving re-usable parts into the core IR.

- The first concern is producing a _valid_ full-path compilation pipeline; optimisation comes second.

- The IR must be scalable to fault-tolerant sizes.

- The IR must be extensible from compiled and interpreted languages, to allow faster prototyping.

- The IR is a base that is used in different ways by different compiler pipelines.
  It will contain efficient representations of certain objects that may not be used in all pipelines.

### Scope of this design

This design is purely concerned with the IR instruction set and data structures.
The associated pass-manager infrastructure and the specifics of particular pipelines are being developed separately.

This design is an MVP.
We expect significant future expansion after implementation.

## IR semantics

This is intended high-level semantics of what is representable in Qiskit MIR.
We don't get into detail of _how_ to represent this yet, only what should be possible.

### IR structure

There are several top-level components to Qiskit MIR.
These include:

- **Instruction list**

  A linearisation of all the instructions in the circuit.
  It supports random access, efficient insertion and removal.

- **Quantum resource table**.
  MIR supports addressing both qubits and abstract groups (see [Quantum types](#quantum-types)).

  "Virtual" groups can be defined on-the-fly by passes as part of the IR, then used as single operands.
  A quantum resource table records the overlaps of groups (approximately: which qubits are in which groups), and can contain custom metadata for each group.

  We expect that one part of a future backend abstraction will be to provide a version of this table for the physical-qubit/-module layout.

- **Symbol table**

  A data structure that links symbol identifiers to information about the symbols, such as their types, whether they are literals (and if so, with what value), and so on.


Notably absent here: we are not defining a control-flow graph or the concept of a "basic block" in the initial implementation.
We expect to add structured control flow over blocks later, but it is not designed in this document.
Some of the above structure may change when we introduce this.


### Instructions

An instruction stores four components:[^instruction-parts]

- **operation**: compile-time fixed information about the action itself.
  This includes some representation of which family of op code (e.g. `pauli_rotation` vs `cx`),
  but may also include operation-specific compile-time fixed information specific to that operation.

- **quantum argument list**: a list of `qid`s that the operation acts on.[^separate-arg-lists]
- **symbol argument list**: the non-`qid` arguments the operation takes.
- **return value**: optionally, the value returned.[^multi-return]

This section does not comment on the implementation of the instruction object, just on the concepts it permits representation of.
The quantum arguments all follow "qubit semantics" (see next section), while the symbol uses and the return follow the semantics of the type of value the symbol refers to.
In this first draft, the only symbol types we define have value semantics.[^memory-semantics]

Qiskit MIR does not enforce SSA form.
Without control flow present, Qiskit MIR can be put into an SSA form (with the `qid` system treated as an extension).
When control flow is added, we will consider then whether to add the necessary phi nodes to continue to allow SSA form.

[^instruction-parts]: We may want to add further metadata/annotations/whatever to individual instructions in the future.
We need to make sure the implementation and APIs don't make this unnecessarily hard.
[^separate-arg-lists]: It would also be a valid design to join the two argument lists and have `qid` be a type of symbol.
Jake roughly prefers the separated form, Kit roughly prefers the joined form, but neither of us have much beyond "it just feels familiar" in mind.
`QuantumCircuit` and `DAGCircuit`'s `Instruction`/`Gate` objects represent a complete "action" that is subsequently applied to "qubits", whereas it's unusual in most (non-quantum) compilers to separate out type of argument from another.
Certainly in the near term, the `qid` argument list represents quite different semantics to anything in the "symbol" list, but that might not always be the case.
[^memory-semantics]: This doesn't preclude us adding the concept of "arrays" or other references to memory in the future.(


### Quantum data model

The quantum data model of Qiskit MIR is abstract, and designed to allow concrete high-level algorithmic languages and concrete backends to model their own semantics.

MIR provides abstractions for quantum data-flow analysis that permits working with ad-hoc groups of qubits, while maintaining the linearity of individual qubits.
It allows multiple simultaneous representations of quantum memory, so high-level algorithms can use a free-form "virtual" layout of their choosing that is progressively lowered to a concrete hardware model.

We introduce three terms:

- **qid**: a "quantum identifier".
  These are the operands of instructions.

- **resource space**: each qid is part of exactly one resource space, and each resource space contains many qids.
  The two initial built-in resource spaces are called "virtual" and "physical".
  We may add more resource spaces in the future[^id-space-expansion].

- **qubit semantics**: the rules for how the underlying "quantum resources" can be modified/copied by instructions.[^qubit-semnatics]

[^id-space-expansion]: We might introduce new resource spaces to represent loops over qubits, or function calls that can be applied to different qubits without instantiating a new function per location.
You don't need this concept in classical computing, where one function call always has the same register uses, but the CPU-register–physical-qubit analogy doesn't hold here.
[^qubit-semantics]: We don't have a full formal specification of "qubit semantics"  yet, but it will follow.
For now, approximately think of them as "a resource that each instruction can use at most once and cannot copy".

#### Qids

An "instruction" in Qiskit MIR takes zero or more `qid`s as arguments.
A `qid` has an optional "qubit width"; this defines how the built-in Pauli instructions act on it.
"Qubit" is not an explicit data type in Qiskit MIR.
A `qid` might directly represent a qubit or a set of qubits, but it is not necessary for the smallest addressable unit to be a qubit.
Every entity in the quantum data model follows "qubit semantics"[^qubit-semantics].

> [!NOTE]
> Jake: I like to think of `qid`s as similar to LLVM's modelling of virtual and CPU registers.
> The analogy is not perfect, and do not assume that any unenumerated CPU-register semantics apply to `qid`, but it may help understand the spirit.

It is allowed and expected to have qubit overlap between different `qid`s used in the program; the set of "contained-in" relations between all `qid`s in the same resource space is an arbitrary DAG.
This allows a backend to define a "module" (for example) that contains other qubits, and have instructions that act on the entire module as a single named entity.
A Qiskit MIR program is invalid if any single instruction acts on two `qid`s that overlap.

The "quantum resource table" of the IR holds information on which `qid`s share references to the same underlying quantum resources.
A `qid` can "contain" other `qid`s.
If all relevant `qid`s have qubit-width information stored, the "contains" relationship includes which qubit indices in the "parent" the "child" refers to.
The complete set of "contains" relationships is an arbitrary DAG, where the nodes are `qid`s and the edges lead from container to contained.
Two `qid`s overlap if any other `qid` is reachable from both in the "contains" DAG.

For example, a bicycle-code hardware backend might allocate a `qid` per individual memory qubit, and a `qid` per complete module.
Each module `qid` is marked in the resource table as "containing" each memory-qubit `qid`, at a specific qubit index.

In order to fully define the data-flow analysis, it may be necessary to define "dummy" `qid`s in a particular resource table that are not valid instruction operands, but represent an overlap between two other `qid`s.

#### Resource spaces

There are two (initial) "resource spaces": virtual and physical.
At any given time, a Qiskit MIR program can use a mix of virtual and physical resources.
An individual instruction may have a mix of virtual and physical operands.

The "memory layout" of the virtual resource space is a property of the IR, and transformations are allowed to add to it, including "adding" new qubits and groups.
This is tracked in the _quantum resource table_ as part of MIR.

The "memory layout" of the physical resource space is a property of the target backend, not owned by an individual MIR program.
The resource table for the physical resource space will likely use the same data structure as the physical one.


### Symbol types

Qiskit MIR comes with several built-in types.
Each non-`qid` parameter in an instruction has one of these types.

- **bool**: a Boolean.
- **Pauli string**: a multi-qubit Pauli operator and phase (integer exponent of `-i`)
- **angle**: a representation of an angle.
  This will store binary fractions of `pi` exactly.

- **quantum state**: the _value_ of a quantum state.
  This is quite unlike `qid`s, which represent some underlying hardware resource (whether it's physical or virtual).
  These are used in the (magic) state-preparation instructions.[^state-symbol]
  These may need to be parametric in some form, to leave open the possibility of (e.g.) `rz(theta)` injection.

This list will certainly expand in the future.

Each of the types named above will come with efficient representations of corresponding literals.

[^state-symbol]: It is still an open question whether the "which state" tracking should be a compile-time attribute of the "state preparation" instruction, or a regular `Symbol` argument.


### Available operations

The purpose of built-in operations is to provide re-usable components that have high memory efficiency and can be acted on with very low overhead.
It must always be possible to have a "dynamic" operations that is defined by the runtime user of Qiskit MIR, to allow extension from outside Qiskit.

Not all pipelines will want to use all built-in operations.
We want to define the set of operations that we expect to both have widespread applicability and be so common that they motivate high efficiency or centralised algorithms to deal with their semantics.

There is a scale of how tightly we define the semantics of built-in assumptions.
The tighter, the more methods and behaviour we can give first-class re-usable support to.
The looser, the more situations the objects are re-usable in.

In all discussion here:

- "Pauli string" is a representation of a multi-qubit Pauli operator.
  We discuss below how Pauli strings relate to their `qid` arguments.

- "arity" means the number of arguments the operations needs.
  We're primarily concerned with the arity of `qid`s and of the implied qubits.

- "variadic" means a list of length that isn't fixed by the op code itself.
  For example, `pauli_measure` with a 12-qubit Pauli string might act on 12 `qid`s of single qubits, or 1 `qid` of a 12-qubit group.
  The number of `qid`s in any given instance of an instruction is known at compile-time; there's no such thing as a runtime list of `qid` in MIR.

The operations set includes:

- `pauli_measure` that takes a Pauli string and variadic `qid`s.
  Returns a bool representing the measurement outcome.

- `pauli_rotate` that takes a Pauli string, an angle, and variadic `qid`s.
  Returns nothing.

- `pauli_project` that takes a Pauli string, a bool, and variadic `qid`s.
  Logically equivalent to `pauli_measure`, followed by a state flip if the measurement result does not match the Boolean argument.
  Returns the result of the intermediate (implicit measurement).[^pauli-project-return]

- `pauli_correct` that takes a Pauli string and variadic `qid`s.
  Equivalent to `pauli_rotate` with an angle of `pi` (and a global-phase correction).
  Intended for representational efficiency.

- `clifford_tableau` that takes some explicit Clifford tableau and variadic `qid`s.
  Returns nothing.

- various explicit low-arity static Clifford "gates" (c.f. [the `stim` operation set](https://github.com/quantumlib/Stim/blob/79ae4f118ca11c615d6d8de7c6eed7d189d3a6eb/doc/gates.md)).
  Each take a number of `qid`s equal to the arity of the gate.
  `qid`s arguments must have a set qubit width of 1.
  Return nothing.

- certain low-arity static non-Clifford gates (e.g. `t`).
  Same considerations on `qid` count and semantics as above.
  Return nothing.

- various low-arity parametric gates (e.g. `rz`), which take the requisite number of angles.
  Same considerations on `qid` count and semantics as above.
  Return nothing.

- state preparation, which takes a variadic number of `qid`s and the state that they are reset to.
  It is an open question whether the state is a compile-time attribute or an argument.[^state-symbol]

- various Boolean-manipulation instructions (e.g. `and`, `xor`), which take two bools and return a bool.

- dynamic extension operation, which will be objects defined externally to the main IR definition (either in Qiskit or elsewhere), and the builr-in IR methods will only interact with via a fixed interface.
  These are for extension types.

Note: we are currently experimenting with particular forms of "predicated" instructions in particular pipelines.
We are not currently in a place where we are confident on the representation or use, so will use the "dynamic operation" form to represent them at first.

[^pauli-project-return]: This does not require that all targets lower `pauli_project` to a literal measure followed by conditional flip, especially if the return value is unused.


#### Pauli-string operations

Operations that take Pauli strings must apply one string to (potentially) multiple `qid`s, where a single `qid` can have a qubit width greater than one.
This requires choosing an "endianness" convention.[^endianness]

Qiskit MIR uses a **little endian** convention.
In an argument list of `qid`s, the zeroth indexed `qid` corresponds to the Pauli terms indexed `[0, width_0)` (where `width_0` is the bit width of the zeroth `qid`), and so on.

A Pauli instruction is not valid if any of its `qid`s do not have a specified bit width, or if the sum of the bit widths of the `qid`s is not equal to the length of the Pauli string.

[^endianness]: It is common in the quantum-computing community to claim that pre-MIR Qiskit had a "little-endian" convention.
This is at best somewhat misleading; pre-MIR Qiskit did not have an endianness convention because it did not previously have any concept of addressing multiple "groups" of qubits simultaneously.
What was typically _meant_ is that Qiskit uses an Lsb0 labelling convention: if `n` qubits were measured into a register and read as an integer value, the bit from the measurement of "qubit 0" is the least significant bit of the value.
All _strings_ were written in this order, including Pauli strings.
Lists of arguments (like Rust `Vec`s, Python `list`s, or parameter lists) used index positions corresponing to qubit numbers.
