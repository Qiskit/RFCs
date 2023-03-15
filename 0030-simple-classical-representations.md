# RFC Title

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com)          |
| **Deprecates**    | None                                         |
| **Submitted**     | 2023-03-15                                   |
| **Updated**       | 2023-03-15                                   |


## Summary

This proposes an experimental representation for operations on classical bits solely within conditions.
This is not required to be the final design for classical handling in Terra, it is meant to be something simple that doesn't entangle existing Terra objects too closely, so can be deprecated in favour of a more complete system in the future.


## Motivation

- Current hardware is gaining support for runtime operations and subsequent conditions on classical bits
- IBM dynamic-circuits hardware can do bit-level calculations, but there is no way to represent this in Qiskit to pass it on
- On Terra, we need more of a feel for what will actually help us represent more complex classical constructs


## User Benefit

- Users of IBM dynamic circuits will be able to take advantage of the classical-processing capabilities of those compilers from within Qiskit
- This support is not limited to IBM so may extend further, but we are not aware of other dynamic-circuits hardware accessible through Qiskit
- For Terra development: this is a "build one to throw away"-type sytem, so we can see in actual usage what works and doesn't for us before we commit to further classical processing.


## Design Proposal

The intent of this RFC is to define an exploratory representation for a simple subset of classical bit-level operations, which expand the current state of Terra's `IfElseOp.condition`.
This representation is not required to be the base for future expansion; it will be permissible to introduce a completely different classical-value type system if that is more appropriate, and deprecate this one.
To that end, this proposal will not include adding any additional public methods to any existing Terra classes that might conflict with an alternative implementation.
In particular, this proposal will not consider adding ease-of-use arithmetic methods to `Clbit` or `ClassicalRegister`, nor will it add expressions that can be used as the parameters to gates.

### Representation

`IfElseOp.condition` and `WhileLoopOp.condition` will expand their allowed values to be an "expression" that evaluates to Boolean.
The allowed expressions will be the following:

- Object:
  - literal `bool` (`True` or `False`);
  - literal `uint[_]` (non-negative Python `int`);
  - runtime `bool` (`Clbit`);
  - runtime `uint[n]` (`ClassicalRegister(size=n)`).
- Unary:
  - `uint[n] -> uint[n]`: `~` (bitwise negation);
  - `bool -> bool`: `!` (boolean negation).
- Binary:
  - `uint[n] * uint[n] -> uint[n]`: `&`, `|`, `^`;
  - `bool * bool -> bool`: `&&`, `||`;
  - `uint[n] * uint[n] -> bool`: `==`, `!=`, `<`, `<=`, `>`, `>=`.

For typing in this initial draft, we will make an exact bijection:

- `ClassicalRegister(size=n)` <=> `uint[n]`
- `Clbit` <=> `bool`

Python Booleans (`True` and `False`) will be used as the `bool` literal values, as they currently are in `Instruction.condition`.
Non-negative Python ints will be used as the `uint` literal values.
These do not have an inherent width.
The width of a literal `uint[_]` will be inferred and fixed to make a containing binary expression validly typed, and mixed-width operations will be forbidden.
A binary expression in this representation _must_ contain at least one type of fixed width; it will be an error to attempt to use the runtime logic on two literals (this simplifies the implementation, and we won't do constant folding).

This is not intended to be a complete set of classical expressions, for simplicity in this initial implementation.
Notably, this MVP will _not_ include:

1. arithmetic on `uint[n]` (`+`, `*`, etc);
2. indexing the bits resulting from one of these rvalue expresions;
3. slicing a `uint[n]` to retrieve a subset of its bits;
4. any method of assigning one of these expressions for reuse elsewhere in the circuit.

Points 1 through 3 are rejected in this initial proposal for the sake of keeping the number of expression-tree nodes and type-system entries small.
Point 4 is rejected because it would require the definition of additional `Operation` objects and potentially modifications to the `QuantumCircuit` class that we are not yet prepared to make.
All of these points are eligible to be included in future work, just not the MVP.


### Construction

#### Stage 1

This is a temporary representation, so it is a non-goal to produce the best possible UX for constructing these objects.
In the initial representation phase, users will be required to build the expression-tree representations themselves.
This will look _something_ like (names up for discussion):

```python
from qiskit.circuit import Clbit, ClassicalRegister
from qiskit.circuit.classical import expr

loose = [Clbit() for _ in [None]*3]
cr1 = ClassicalRegister(2)
cr2 = ClassicalRegister(2)

# Equivalent to the current condition `(cr, 2)`:
equal_reg = expr.Equal(expr.Value(cr1), expr.Value(2))

# Equivalent to the current condition `(loose[0], False)`
equal_bit = expr.Equal(expr.Value(loose[0]), expr.Value(False))

# The above example, but using a Bool-valued unary expression:
not_bit = expr.Not(expr.Value(loose[0]))

# The comparison 0 < cr1 <= cr2
bounded_reg = expr.And(
    expr.Less(expr.Value(0), expr.Value(cr1),
    expr.LessEqual(expr.Value(cr1), expr.Value(cr2),
)
```

These expressions will then be given directly to `IfElseOp` or `WhileLoopOp` in their `condition` fields.
These expressions will not be valid in the general `Instruction.condition` field; we do not want to expand any support for something we are already moving to remove.


#### Stage 2

Looking beyond the initial phase, it will be convenient to define a less verbose way for users to create these expressions.
Since this document is intended to define a representation that can be removed and deprecated from Terra easily if it needs to evolve, we will _not_ consider overloading the Python magic-methods `__and__` etc on `Clbit` and `ClassicalRegister` in general Terra usage.
However, we can do so in a limited scope, which will temporarily change the semantics.
This will look like (continued from above code block, and the same examples in the same order):

```python
with expr.build():
    equal_reg = cr1 == 2
    equal_bit = loose[0] == False
    not_bit = ~loose[0]
    bounded_reg = (0 < cr1) & (cr1 <= cr2)
```

It is not possible to overload the behaviour of the Python Boolean operators (`not`, `and` and `or`) for classes, which is why the bitwise operations `~` and `&` are used instead (with the corresponding precedence frustrations).
Within the `expr.build()` context, the magic methods of `ClassicalRegister` and `Clbit` will be temporarily overridden to be an eager builder interface for the syntax tree given above.
This means that the expressions will not need to be bound to variables in order to use them, as the resulting objects will be well-defined and the correct tree form.
For example, it will be valid to do

```python
qc = QuantumCircuit([Qubit()], loose)
with expr.build():
    with qc.if_test(~loose[0]):
        qc.h(0)
```

The reason to require entering the builder context to make the magic methods available is for potential future ease of change/removal/deprecation.
The mutation of global state required to do this makes the builder interface non-thread-safe, but this is not judged to be a problem; it is uncommon to attempt to multithread during manual circuit construction, and the advantages of easy removal/representation change completely outweigh the potential concern.


## Detailed Design

The new functionality will live in in `qiskit.circuit.classical`.
The expression-builder objects will all be exposed from `qiskit.circuit.classical.expr`, to allow people to bring the desired values into scope either with a namespace prefix `expr` or by a star-import of limited scope.

> From this point on, I will omit the prefix `qiskit.circuit.classical.`.

The class `expr.Expression` will be an abstract base class with two read-only fields:
```python
# file: qiskit/circuit/classical/expr.py

class Expression(ABC):
    resources: typing.Set[Clbit]
    type: qiskit.circuit.classical.Type
```

The `resources` field is effectively a `use-def` tracker, which will more easily enable the conversion to data-flow (`DAGCircuit`) format to evaluate the necessary wires of an instruction.
The `type` field is a resolved type of the contained expression.
In this initial release, the only time that `type` will not be a fully qualified type will be when representing a `expr.Value` of a `uint` literal.
Type errors will raise immediately on attempted construction.
In the future, we may consider relaxing this restriction and having a type-evaluation pass that runs on the complete `QuantumCircuit` AST representation, but that is for later work.

The expressions in the above examples will be final (uninheritable) subclasses of `Expression`:
```python
class Value(Expression):
    value: Any

class Less(Expression):
    left: Expression
    right: Expression

...
```

Similarly, the type system will be represented by values of the type `types.Type`:
```python
# file: qiskit/circuit/classical/types.py

class Type(ABC):
    pass

@typing.final
class Bool(Type):
    pass

UNKNOWN = object()

@typing.final
class Uint(Type):
    size: int | Literal[UNKNOWN]
```


## Implementation plan

PR 1:
- write representation of `Expression` and `Type`
- test that these objects can be constructed and handle type checking correctly

PR 2 (depends on 1):
- add support to `Expression` of evaluated type `Bool` to `IfElseOp.condition` and `WhileLoopOp.condition`

PR 3 (depends on 2):
- add support for exporting these `Expression` conditions to the OpenQASM 3 

PR 4 (depends on 2):
- add support for these `Expression` conditions into the `DAGCircuit` <-> `QuantumCircuit` conversions (i.e. enable the transpiler to work with them)

PR 5 (depends on 1):
- add the `expr.build()` context manager

Pull requests 1 through 4 are required for minimal support in a Terra version.
PRs 2 and 3 are semi-parallelisable; the export support can be made prospectively assuming that the `condition` field might be an `Expression`, but the final tests will need PR 2 merged first.
Pull request 5 can be permitted to follow in a subsequent Terra release, if necessary, despite being parallelisable with all PRs following 1.


## Alternative Approaches

### AST-visitor builder interface

The stage-2 builder interface for the expression syntax given above had a problem that Python does not allow classes to override the behaviour of binary Boolean operations.
This could be avoided instead if we had the builder interface be something like
```python
@expr.build
def _equal_reg(cr: ClassicalRegister):
    return cr == 2

@expr.build
def _bounded_reg(cr1: ClassicalRegister, cr2: ClassicalRegister):
    return 0 < cr1 <= cr2

equal_reg = _equal_reg(cr)
bounded_reg = _bounded_reg(cr1, cr2)
```

This is possible with function decorators because the decorator can access the Python AST of the contained code and rewrite it.
`with` statements still execute, we can just control the context within that happens.

This function-decorator method is not chosen because:

- in practice, since it only applies to expressions and is not part of a larger circuit builder interface, it ends up being much more verbose due to the need to first define, then call
- it is harder to write and maintain a Python AST visitor, which makes it less ideal for a preliminary implementation
- it has to do type checking twice; once during the AST visit, and once during the subsequent object call


## Questions

?

## Future Extensions

- An entire classical system including the ability to assign runtime expressions to variables that are managed by `QuantumCircuit`.
- An extension of the type system to include values that can be used as the operands of gate objects.
