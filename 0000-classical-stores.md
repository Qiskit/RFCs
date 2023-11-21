# Classical storage and load instructions

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com)          |
| **Submitted**     | 2023-09-07                                   |
| **Updated**       | 2023-11-13                                   |


## Summary

In scope:

- Add new, block-scoped classical `Var` storage locations to `QuantumCircuit` that will (long term) replace `ClassicalRegister` and `Clbit`.
- Introduce instructions to manually assign `Expr` nodes to a storage location.
- Allow input and output from these storage locations at the start/end of program execution.

Out of scope:

This RFC does not propose making `Expr` nodes available in any more places than they already are, nor adding any new types to the system.
In particular, this RFC does not yet address the question of making `Expr` nodes usable in `params` locations (such as the angle inputs to gates).


## Motivation and Benefit

Qiskit needs a way to represent the sorts of "parametric update" that we expect hardware vendors to be implementing in the near future.
For example, we will ideally be able to use a single compiled circuit (both in the Qiskit and hardware senses of the term "compiled") that accepts certain parameters as inputs at runtime, which may affect conditional expressions or gate angles.
This RFC only addresses the question of defining new classical-variable storage locations and writing to them for Qiskit within its existing typed `Expr` system, which is currently limited to unsigned integers and Booleans and cannot be used for gate angles.

For example, Pauli-twirling workloads want to do something like
```python
from qiskit.circuit import QuantumCircuit
from qiskit.circuit.classical import expr

qc = QuantumCircuit(2)
with qc.if_test(expr.lift(TWIRL_BITS_0)):
    qc.x(0)
with qc.if_test(expr.lift(TWIRL_BITS_1)):
    qc.x(1)
```
where `TWIRL_BITS_*` are some values defined by the particular set of twirls to be done, and are known before the circuit begins execution, but not _necessarily_ at the time of the definition of the circuit, or the circuit should be run with many different values for them.
These may also want to be compared to the results of mid-circuit measurements, or have arbitrary run-time classical expressions built from them.
Right now, Qiskit provides no way to parametrise `if` conditions in any form; even the compile-time rewriting of `assign_parameters` will not support them, because `ParameterExpression` objects are (deliberately) not valid in `condition` fields.

Similarly, a user may need to calculate a value based on some measured bits, potentially dependent on some `if` condition, and store the value in classical memory for later use in the circuit.
In other words, they need variable assignment at runtime.


## Design Proposal

At a high level, this proposal concerns adding concepts equivalent to the OpenQASM 3 `input` (and potentially `output`) and variable-assignment-by-expression statements to Qiskit, within the classical types we already support.
I do not yet propose removing `Clbit` or `ClassicalRegister`, nor trying to change them into some other form; for the time being, they retain their "special-case" status.

The additional user interface for defining and working with variables on `QuantumCircuit` is:

```python
class QuantumCircuit:
    def __init__(
        self,
        ...,  # existing structure
        *,
        inputs: Iterable[expr.Var] = (),
        captures: Iterable[expr.Var] = (),
        declarations: Mapping[expr.Var, expr.Expr] | Iterable[Tuple[expr.Var, expr.Expr]] = (),
    ): ...

    def add_var(self, name_or_var: str | expr.Var, / initial: expr.Expr) -> expr.Var: ...
    def add_capture(self, *vars: expr.Var): ...
    @overload
    def add_input(self, name_or_var: str, type: types.Type, /) -> expr.Var: ...
    @overload
    def add_input(self, name_or_var: expr.Var, /) -> expr.Var: ...

    def store(self, location: expr.Expr, value: expr.Expr, /): ...

    @overload
    def get_var[T](self, name: str, *, default=T) -> Union[expr.Var, T]: ...
    @overload
    def get_var(self, name: str, *, default=NotGiven) -> expr.Var: ...
    def has_var(self, var: Union[str | expr.Var]) -> bool: ...

    # All variables, including inputs and captures.
    def iter_vars(self) -> Iterable[expr.Var]: ...

    # All do what they say on the tin.
    def iter_declared_vars(self) -> Iterable[expr.Var]: ...
    def iter_input_vars(self) -> Iterable[expr.Var]: ...
    def iter_captured_vars(self) -> Iterable[expr.Var]: ...
```

`QuantumCircuit.add_var` and `.add_input` are for declaring a variable with an owned storage location, the difference being that `add_var` is an internal variable and requires an initialiser, while `add_input` marks this as in input variable to the circuit and cannot have an initialiser.
The variable remains in scope for the whole body of the circuit.
Variables are not automatically inherited by nested scopes; they must be explicitly closed over when building an inner scoped `QuantumCircuit` instance using `QuantumCircuit.add_capture`.
Only `add_input` takes a `Type`, and even then, only if the input is given as a `str` name.
All other forms read the type from the `expr.Var` or the initialiser `expr.Expr`.
It is an error if the initialiser expression is not of a type that implicitly casts to the type of the `Var`.
If a `str` name is given in `add_var`, the type of the resulting expression will be exactly the type of the initialiser, and this should be manually cast to the desired type if it is not the exact required type.

The initialiser of `QuantumCircuit` is expanded to allow setting these as well.
This is not intended to be how the _first_ circuit a user creates constructs its variables in general; it is meant to make it easy to build more circuits similar to the first circuit by reusing the same `Var` instances that are already held.
In the initialiser, all the variables must be given as `expr.Var`, since if a string was permitted, a user would then have to do `get_var` to retrieve the `Var` before they could do anything anyway, at which point they can use `add_var` which is just as ergonomic.
The keyword `declarations` must take some form of mapping from the `expr.Var` values to the initial expressions.
A user must supply a mapping or iterable that will yield the `Var` nodes in a suitable order for declaration; it will be an error if an initialiser expression relies on a `Var` that has not yet been seen.
Python's built-in `dict` yields its items in insertion order (up to removals), so this is not expected to be an onerous ask for the API.

`QuantumCircuit.store` inserts a `Store` instruction, which at QPU runtime evaluates `value` and stores it in the memory location that `location` resolves to.
The `location` must evaluate to a writeable memory location.
In this initial RFC this is limited to `Var` instances, though in the future one can imagine some `expr.Index` node or the like existing, for runtime indexing into objects that are array-like in bits.

`QuantumCircuit.get_var` and `.has_var` are convenience methods for the user.
`.has_var` is a simple Boolean checker akin to `dict.__contains__`.
`.get_var` is akin to `getattr`; if the default is not given it raise an error, while if a default is given, that is returned instead.
This design is to make it ergonomic both when all variables are expected to be present (one need not check manually that an actual `Var` was returned; it's sufficient to _not_ catch the exception), and when an action should be taken _only_ if the variable is present, such as in
```python
if (var := qc.get_var("a", None)) is not None:
    # Do something with `var`, which is guaranteed to be a `Var`,
    # but no extra `try`/`except` is needed if it might not exist.
```
These methods are present to solve a common user paper-cut when working with `Parameter` values, where they can't retrieve the parameter only by its string name despite circuits enforcing that these instances have unique names or are referentially identical within a circuit.

The iterator methods `iter_vars`, `iter_declared_vars`, `iter_input_vars` and `iter_captured_vars` all return single-use iterator objects over the relevant variables.
This choice lets `QuantumCircuit` use a mutable type internally for performance when adding new variables, while allowing iteration through the variables without the necessity of copying the collection when outputting it to the user to avoid them messing with it.


### Examples

The example given above in the motivation section would look like:

```python
from qiskit.circuit import QuantumCircuit
from qiskit.circuit.classical import expr, types

qc = QuantumCircuit(2)
twirl_bits_0 = qc.add_input("twirl_bits_0", types.Bool())
twirl_bits_1 = qc.add_input("twirl_bits_1", types.Bool())
with qc.if_test(twirl_bits_0):
    qc.x(0)
with qc.if_test(twirl_bits_1):
    qc.x(1)
```

The result of `qasm3.dumps()` on this circuit will be (subject to header-file discussions, etc):
```qasm3
OPENQASM 3.0;

include 'stdgates.inc';

input bool twirl_bits_0;
input bool twirl_bits_1;

qubit _qubit_0;
qubit _qubit_1;

if (twirl_bits_0) {
    x _qubit_0;
}
if (twirl_bits_1) {
    x _qubit_1;
}
```

---

Using the control-flow builders will allow scoped declarations, and using a variable declared in an outer builder scope will automatically cause it to become captured within the output.
For example:

```python
from qiskit.circuit import QuantumCircuit, QuantumRegister, ClassicalRegister
from qiskit.circuit.classical import expr, types

qr = QuantumRegister(3)
cr = ClassicalRegister(3)
qc = QuantumCircuit(qr, cr)
# The explicit `type` is the same as what would be inferred here so not
# technically necessary, but illustrative of the capability.
mask = qc.add_var("mask", expr.lift(5, type=types.Uint(3)))
qc.h(0)
qc.cx(0, 1)
qc.cx(0, 2)
qc.measure(0, 0)
qc.measure(1, 1)
qc.measure(2, 2)

# 'with_mask' is defined now with a value that's only known after the measures.
with_mask = qc.add_var("with_mask", expr.bit_and(mask, cr))
with qc.if_test(expr.equals(with_mask, mask)):
    scoped = qc.add_var("scoped", expr.logic_not(cr[1]))
    with qc.while_loop(scoped):
        qc.measure(1, 1)
        # The control-flow builder automatically captures `scoped` into the while-loop body.
        qc.store(scoped, expr.logic_not(cr[1]))
# 'scoped' is no longer in scope here, and attempts to use it in the circuit will error.
```

This produces the (fairly useless) OpenQASM 3 program:
```qasm3
OPENQASM 3.0;

include 'stdgates.inc';

qubit[3] qr;
bit[3] cr;

// Both of these are declared immediately but initialised at the relevant point in the circuit.
bit[3] mask;
bit[3] with_mask;

mask = 5;
h qr[0];
cx qr[0], qr[1];
cx qr[0], qr[2];
cr[0] = measure qr[0];
cr[1] = measure qr[1];
cr[2] = measure qr[2];
with_mask = mask & cr;
if (with_mask == mask) {
    bool scoped = !cr[1];
    while (scoped) {
        cr[1] = measure qr[1];
        scoped = !cr[1];
    }
}
```

I don't expect the declaration of scoped variables to be valid for execution on hardware immediately, but I think it's a little easier to implement in Qiskit if the possibility is always there.

The control-flow builder usage in this case would desugar into this long-form construction:
```python
from qiskit.circuit import QuantumCircuit, QuantumRegister, ClassicalRegister
from qiskit.circuit.classical import expr, types

qr = QuantumRegister(3)
cr = ClassicalRegister(3)
qc = QuantumCircuit(qr, cr)
mask = qc.add_var("mask", expr.lift(5, type=types.Uint(3)))
qc.h(0)
qc.cx(0, 1)
qc.cx(0, 2)
qc.measure(0, 0)
qc.measure(1, 1)
qc.measure(2, 2)
with_mask = qc.add_var("with_mask", expr.bit_and(mask, cr))

true_body = QuantumCircuit([qr[1]], [cr[1]], captures=[mask, with_mask])
scoped = true_body.add_var("scoped", expr.logic_not(cr[1]))

while_body = QuantumCircuit([qr[1]], [cr[1]], captures=[scoped])
while_body.measure(qr[1], cr[1])
while_body.store(scoped, expr.logic_not(cr[1]))

true_body.while_loop(scoped, while_body, [qr[1]], [cr[1]])

qc.if_test(expr.equals(with_mask, mask), true_body, [qr[1]], [cr[1]])
```


## Detailed Design

### Modifications to `QuantumCircuit`

In addition to the public API surface discussed above, `QuantumCircuit` will need to keep a track of its variables internally.
The public-access `iter_*` methods are designed to allow `QuantumCircuit` to use private mutable, growable data structures to do this efficiently.

The purpose of having explicit variable capture is such that it remains impossible for a `QuantumCircuit` to be in an invalid state, and to make explicit the requirement to use the same variables.
We currently have a problem in the control-flow operations where the exact classical registers must be used in inner blocks (if they are to be accessed), and this constrains the data in the `Clbit` binding between the `CircuitInstruction` and the individual blocks' `.clbits` in a way that is not clear at all, and introduces many possibilities for data incoherence.
Requiring closures to be explicitly stated sidesteps all of this; there is no longer a "binding" step, and the validity of all the operations can be enforced during construction only.
Holding a `QuantumCircuit` then contains all the information needed to work with the new objects within it.

Of existing circuit methods:
- `QuantumCircuit.copy_empty_like` would need to expand to include variables in the copy.
- `QuantumCircuit.from_instructions` needs to gain the same keyword arguments as the `__init__`.
- `QuantumCircuit.compose` and `QuantumCircuit.tensor` certainly need to have _some_ handling for this (even if it's just erroring if both circuits contain variables), but it's not clear to me how variable collisions should be handled.
  See the open questions section.
- The control-flow builders all need to learn to capture these variables when used in enclosing scopes, to intercept declarations to add to the built scope instead of globally, etc.
- For now, I propose that we make `to_instruction` invalid for circuits involving classical parameters.
  There does appear to be a clear definition for these (subroutines), it's just a large extension that I want to leave to the future.

When an `add_var` (which takes an initialiser expression) is added to the circuit, the declaration of the space is assumed to happen at the top of the scope, but the variable is initialised by a `Store` at that particular point in circuit execution.
This allows a variable to be initialised with data that only becomes known during the circuit execution, but does not permit shadowing variables declared in the same scope.
It's easier to track without it, less confusing, and same-scope shadowing isn't permitted in OQ3 anyway.

### Modifications to `expr.Var`

The name of a variable should at least be retrievable from a `Var`; allowing this is necessary to make much of the API more convenient for the user.

Currently, `Var.var` stores the `Clbit` or `ClassicalRegister` that the variable wraps, as a way to retrieve the Qiskit "storage location" from the variable.
It's not yet clear to me where this "storage information" requirement will be needed within Qiskit (which does little classical processing) except in the case of wrapping an existing `Clbit` or `ClassicalRegister`.
I can't fully foresee at this point what information should be tracked in the `Var`, or if the precise reference to the `Var` should be sufficient (as is the case for `Clbit`, for example); I think this might become clearer during implementation.

I do believe, though, that it would be wrong to allocate `Clbit` instances to back each of the "bits" of a typed classical variable; I wrote more about this in "alternative implementations" below, and I seriously considered this when writing this RFC.
I suspect that there's some argument to be made around making sure the circuit-copy API allows the precise same `Var` instances to be used in more than one circuit so that expressions can be transplanted from one to another, such as from iterating through a circuit to build up another without recursing all the way through the `Expr` tree to rebuild expressions.

In general, we do not need and cannot expect a `Var` instance to be reproducible from its human-readable identifier alone, because a `Var` refers to a particular storage location in a particular program (this is the reason for `QuantumCircuit.get_var` and friends).

For the initial implementation, I propose to _not_ implement equality for `expr.Var` other than referential equality.
It's not clear to me that any context-free equality between two `Var` instances can make sense.
The structures for comparing semantic equality of variables is already in place within the `structurally_equivalent` visitor, and the key functions there can be used to implement a sort of typed de Bruijn index comparison.
The trouble with implementing equality for `expr.Var` instances is that a context-free implementation could easily allow spurious equalities between variables of the same type and name declared in different scopes; these are _not_ equal.


### Changes to `DAGCircuit` tracking

We do not use `Clbit` backers for each `Var` in part so that we do not need to track each `Clbit` separately.
We will need to add additional tracking edges for these `Var` nodes, but in general there is no need for them to use the same "wire" form that `Clbit` previously did.
That made sense for `Qubit` tracking, where there's no distinction between "read" and "write" operations, only uses, but a variable is typically read far more than it's written, and we do not want to add spurious data-flow restrictions between operations that all just read from the same memory without writing to it.

Instead, we can insert a directed edge from a write to the variable to all the reads that it is live for.
On the next write, including if a control-flow block may conditionally write to a closed-over variable, all the reads add an edge point from themselves to that write, the original definition becomes dead, and the new write becomes the source of wires to subsequent uses.
Loop bodies do not need a self wire here; it's not needed for the data-flow tracking that Qiskit is concerned with.

This is a crude approximation at proper SSA tracking of variables, but this should be sufficient, if a little over-cautious; certain assignments in deeply nested conditions could introduce ordering relations that could be somewhat simplified.
In general, though, there's only so much Qiskit should attempt at this point.


### The `Store` instruction

`QuantumCircuit.store` is a shorthand for emitting a `Store` subclass of `Instruction`.
For further discussion on why this is a subclass of `Instruction` and not a more fundamental modification to `QuantumCircuit.data` or `CircuitInstruction`, see the "alternatives" section below.

The two parameters of `Store` are the location and the value.
These are stored in `Store.params`, like a parametrised gate object, so that they can be moved to a more suitable location in a similar manner in the future.
It is not expected that any code that currently inspects `Instruction.params` should be able to automatically work with `Store`; these objects require special handling, but this is generally true for the `.params` field which is (unfortunately) quite unconstrained in its typing.


## Alternative Approaches

### Large-scale alternatives

#### Expand `condition` or `Expr` fields to support `ParameterExpression` and use `assign_parameters`

As discussed in [RFC 10](0010-simple-classical-representations.md), reuse of `ParameterExpression` is not suitable for typed inputs.
It also fundamentally represents a symbolic calculation on real numbers, rather than a circuit run-time evaluation of an object, so it would be close to impossible to align the two computational models.

#### Give `Var` nodes backing `Clbit`s

I seriously considered this to the degree of holding up publishing this RFC to pivot to this design.
It has the benefit of:
- making `ClassicalRegister` a little less special-cased; it's a variable with contained bits.
- it would be possible to represent a `measure` directly into a bit of a new-style variable

However, in the end, I have many reasons that I think this would not the correct decision:

- There's 5 pointers of storage space required per Python-space `Clbit` that's just wrapped by a `Var`, not to mention the additional costs of tracking the bit within `QuantumCircuit`.
  For a `double`, that's over 2kB of extra tracking, even if we do nothing with it.
- It implies that we need to track each of these `Clbit` wires separately in the DAG, even though the data-flow is bundled - again, lots of unnecessary memory use.
- There is a natural split in hardware between a `Clbit`, which is a target for a `measure` instruction and is likely written to in controllers very close to the QPU hardware, and classical variables that can be in outer control systems.
  For example, gathering the result of a `measure` into the particular bit of a `uint[8]` might involve more expensive transfer and synchronisation operations than adding together two `uint[8]` objects.
- If there are `Clbit` instances backing `Var`, we presumably would need to duplicate this idea of data-tracking into the `.clbits` fields of `CircuitInstruction` and `DAGOpNode`, which means we need to define an "ordering" to how the bits are encountered through every instruction, and because they would automatically be in the `Expr` tree, doing so introduces the possibility of failed data coherence, where the `.clbits` fields do not match the nodes.
- If we have `Clbit` instances, we cannot use the same methods of variable-closure in nested scopes, and compound instructions taking input parameters or using input parameters only internally would have confusing behaviour between the `.clbits` fields of their own `CircuitInstruction` and their internal `Var` nodes.
- `ClassicalRegister` would _still_ be special cased, however, because it does not fundamentally "own" its bits; it's possible to have aliased bits in more than one register.
  This is very fiddly to work with in general, is highly unlikely to have any real hardware support in the near term, and it's unclear how it's useful.

#### Give `Store` first-class support as an alternative to `CircuitInstruction`

The RFC currently proposes to jam `Store` into being a subclass of `Instruction` that has inherent classical bits (similar to `Instruction.condition` or the `ControlFlowOp` subclasses).
This means that all consumers of `QuantumCircuit` that care about data-flow orderings and display must know to check for this particular subclass, and include custom code to handle this.
Similarly, the `DAGCircuit` would need to grow an alternative to `DAGOpNode` that could represent this with new first-class support.

A strong alternative would be to have different versions of the `CircuitInstruction`/`DAGOpNode` circuit context object that better represent the "special" classical operations such as `Measure`, `Store` and the `ControlFlowOp` subclasses.
This does not inherently change that the user might forget to handle these cases, but changing the structure of the `CircuitInstruction` objects as they appear in Python space offers us a way to make them fail loudly, not silently ignore data dependencies.
In Rust space, we can use the algebraic type system to more easily enforce that all cases are handled.
I think this is a direction we absolutely must take in the future, but that the time to do that is once the backing data of both `QuantumCircuit` and `DAGCircuit` is stored in Rust space and the transition path is easier.
If we were to do this right now, all consumers of `QuantumCircuit.data` would be broken, and this would either require an immediate huge amount of work to change how `Measure` and the control-operations are stored in Qiskit, or we would be adding an even more special case that `Store` already is in that it would not just be a special `Instruction`, but also a special `CircuitInstruction`.
I propose that we revisit the internal representation of all these classical operations in one unified redesign for both `QuantumCircuit` and `DAGCircuit`, rather than trying to do it on-the-fly.


### Alternatives to smaller details

#### Automatic closure over variables declared as `input`

In the current system, it is not valid to have a circuit that has both closure variables _and_ input or output variables.
Instead of having separate `add_input()` and `add_capture()` methods, one could imagine that inner scopes could `add_var` variables to be either of type "input" or a new type "closure".
The reason to choose to have a separate method for these semantics is that the signature of the `add_var` method would become quite unpleasant if it needed to do all these things; it already has multiple signatures because it needs to take either `(var,)` or `(name, type)`.

It is not possible to close over a variable with just its name and type, because we'd also need the information about which root allocation it refers to build up the `DAGCircuit` data-flow graph, and we don't gain that information until the circuit is embedded within another.
This means that `add_var` would need to have two forms:
- one that looks like `add_var(name, type, "input")`
- one that looks like `add_var(var, "input")` or `add_var(var, "closure")`
This is awkward to produce a coherent type signature for that allows sensible named-keyword calling conventions (in places where we want to allow that), which increases mental load of the user.
Further, it makes `add_var` perform two logically separate actions: declaration of a new storage location, and re-use of an existing storage location.

Finally, the potential mental load from having two separate methods is mitigated because we do not expect users to manually call `add_capture` themselves.
The control-flow builders can intercept all `expr.Expr` values coming in, and automatically cause the scope under construction to close over any undefined variables, if they exist in the outer circuit.
`QuantumCircuit.add_capture` would only be necessary when manually constructing a `QuantumCircuit` scoped block separately, where the user must manage all the context themselves.
This is behaviour libraries may choose to do, but we already _strongly_ suggest that users use the control-flow builder interface.


#### Allow `str` names in `expr` builder constructions instead of using `QuantumCircuit.get_var` to get there

It feels tempting to allow things like `expr.bit_and("a", 7)` to automatically promote `a` to a `Var`.
This primarily fails because we cannot in general correctly infer the type of the resulting `Var`, which would require a sufficiently awkward type signature like allowing `Tuple[str, Type]` as the input in any position.
The work that `QuantumCircuit` would need to do when accepting any `Expr` would also become more complex, as it would need to rewrite expressions to use the correct instances of `Var`, or fill in the required storage locations (`Var.var`), depending on what we put in `Var.var`.

The problem with `Parameter`'s inequality to its name is not typically a desire to use the name in symbolic expressions (analogous to the `expr` builders here), which users generally understand _must_ use the `Parameter` instance in order to produce a `ParameterExpression`, but simply in retrieving the parameter instances to later bind values to.
`QuantumCircuit.get_var` solves the latter real-world problem, without introducing awkwardnesses into the type signatures of other functions.


#### Allow `Measure` to write into `Expr` l-values like `Store`

This RFC currently has a distinction between `Measure` which still uses the old `cargs` form and consequently writes into a `Clbit` instance, and `Store` that contains the memory locations inherently, and thus must be extracted by things like `DAGCircuit.apply_operation_back` (and would be easy to miss).
The main reason not to expand `Measure` directly in this RFC is the complexity; `Measure` is an _exceptionally_ common operation (perhaps the only one that is almost guaranteed to be in a complete quantum program), and it's not yet clear what the full expansion path here should be.
Allowing a direct measurement into an `Expr` would involve updating every consumer of `CircuitInstruction.clbits` to know that the values may not always be `Clbit` instances anymore, which would be too breaking a change.
For more discussion of potential ways in which `Measure` and `Store` may begin to look different in the future, see the "Give `Store` first-class support" heading.

With the behaviour described in this RFC, the way to store a measurement result to an arbitrary `Expr` will be
- measure into a `Clbit`
- use `QuantumCircuit.store` to copy the measured value into the required place

This is more work for users, but is not unreasonable from the perspective of real hardware, where the locations that qubit measurements are stored into typically need more data transfer and synchronisation to be brought into the places where they can have classical computation done with them.
The current design does not preclude us changing this in the future.


## Questions

### Should we add a `params` field to `CircuitInstruction` now or later?

This RFC currently proposes to keep the `params` of `Store` inherent to the object, much like they are with `RZGate` and other parametrised operations.
This prevents `Store` from being a singleton, and in this case, means that there's more special-case handling needed.

In this RFC, I went with keeping the parameters inherent to `Store` rather than attempting to thread through a `params` field into `CircuitInstruction`.
I'm worried that trying to do too much there could interact very oddly with the `.clbits` field, and would lead to an odd split where some things were parametrised in `CircuitInstruction` and others not.
Until it's clear how _all_ instructions should be parametrised, I elected to keep the new object in the same sort of special case as the other classical operations (the control-flow operations).


### What does the API for `QuantumCircuit.compose` expand to?

`QuantumCirucit.compose` is in effect an inlining operation.
It's not clear to me whether it should be possible for a circuit to involve variables whose names conflict in scope with the program it's being inlined into, or what if any automatic variable-renaming methods we should make available.

If there are no conflicts in the variable names, it seems expected to me that the lists of input variables would catenate, and so on.
Similarly, two declared `Var` instances that conflict (either by having incompatible initialisers, or one being an input, one not, etc) cannot be allowed to coexist, but does this mean that the composition should be rejected, or should we implement a renaming?

For now, we will impose "error on conflict", and revisit if a compelling and safe use-case to do otherwise appears.


### Is it necessary to add an `expr.Index` immediately?

At the moment, it's inconvenient to get a measurement result from a `Clbit` into one bit of a declared classical variable; it effectively needs to be done as
```python
with qc.if_test(bit_0) as else_:
    qc.store(my_var, expr.bit_or(my_var, 0b0001))
with else_:
    qc.store(my_var, expr.bit_and(my_var, 0b1110))

with qc.if_test(bit_1) as else_:
    qc.store(my_var, expr.bit_or(my_var, 0b0010))
with else_:
    qc.store(my_var, expr.bit_and(my_var, 0b1101))

# ... and so on.
```

This is not exactly efficient, nor convenient.

Much of this can probably be avoided by judicious use of `ClassicalRegister` instead stores into other variables, but it's worth bringing up as a question now.
`ClassicalRegister` would not be fully suitable when only part of an object needs filling with the results from bits.

In general, one can imagine having an `expr.Index` expression that produces bit-typed l-values, and allowing stores into that.
Currently this RFC does not propose this to avoid scope creep, but I don't know if it's something that ought to be done now.


## Future Extensions

### Removal of `Clbit` and `ClassicalRegister`

The long-term goal of this work is to completely replace the classical-memory handling of Qiskit.
This means the final removal of `Clbit` and `ClassicalRegister`, so that `Var` is the only representation of owned classical memory.

A core problem with `ClassicalRegister` is that is _does not_ represent owned memory; registers are permitted to alias each other.
This is not much of a problem for quantum registers, since those are purely a programmer convenience and have no semantics within Qiskit's compiler (they're completely erased during the lowering to hardware).
However, for classical registers, the underlying classical data is directly writable, copyable, and may form part of the returned outputs, which makes the bit-level aliasing hugely difficult.
This is more difficult even than for the handling of aliasing pointers in standard classical computing (though Qiskit has no pointer type, and is not intending to add one), because the aliasing is at the level of bits, which may not even be contiguous in memory; they are smaller than the smallest addressable unit in standard controller memory.

The new typed classical data represented by the `Var` node with (e.g.) `Uint(8)` is intended to completely replace `ClassicalRegister` in the long term.
This replacement is not being done in this RFC for a few reasons:

- `ClassicalRegister` is heavily used in user code, and the transition will be difficult to orchestrate.
- The "output" format from hardware backends still assumes `ClassicalRegister`; bitstrings are returned with spaces in them denoting the splits between classical registers (they assume no aliasing).
- This RFC is trying to keep its scope manageable.

The intent is to remove `ClassicalRegister` at a later date, but not for the Qiskit 1.0 release.
Further work on making sure the `Expr` tree can represent all uses of `ClassicalRegister` must come first, as must work to ensure that a new, typed system of variable outputs from quantum programs is well defined, on the path to implementation by hardware, and supported by Qiskit (see next extension).

### Add support for circuit outputs

OpenQASM 3 supports an `output` specifier on variables for rich returns from programs.
Qiskit currently doesn't have a way of representing this through its result types, and it's not clear when hardware vendors will have support for this.
For ease of user expectations, it's left til later.

### Constant folding akin to `QuantumCircuit.assign_parameters` for `Parameter`

A user may desire to take a circuit from being parametrised in terms of input `Var`s, provide concrete values for some of these, and retrieve a new circuit that does not have these same variables.
This is approximately akin to `QuantumCircuit.assign_parameters` for `Parameter` instances, except that `Parameter` comes with an expectation that this operation _must_ be permitted, because they were originally always user-side objects only.

This RFC does not include a method to do this; in general, I think its omission will help define the distinction between how `Parameter` and `Var` should be used (certainly in the near term) as user-side compile-time and QPU runtime objects respectively.
It also gets into the realm of classical-compilation optimisations that we are saying are out of scope of Qiskit; implementing the full folding would require us putting in place the mathematics to support all the types of the type system, and effectively requires us to implement an interpreter for the mini-language.


### Use an SSA form for `DAGCircuit` classical variables

We currently consider classical optimisations out-of-scope for Qiskit, but were we to do them, we might consider using an SSA form for the `DAGCircuit` classical-variable tracking rather than representing stores as writing into the same variable.
I did not want to get into such large rewritings of how we handle _all_ classical data in Qiskit, in part because I think our (especially `DAGCircuit`) representations of control-flow are unsuited for this at the moment.


### Add support for `QuantumCircuit` to represent a hybrid quantum--classical subroutine

The addition of `input` variables to circuits means that they can now begin to represent much of what OpenQASM 3's `def` subroutines can.
This is a clear possibility for extension in the future, but with limited-to-no hardware support currently planned, I don't think this is something for now.

It's more of a priority to get some method to expose `extern` classical operations into the type system to better take advantage of real-world capabilities.


### Make the returned variable from `QuantumCircuit.for_loop` a `Var`

This is logically what it should be, but until we have support for using `Expr` nodes as gate parameters, it would make `for_loop` (even more) useless.
