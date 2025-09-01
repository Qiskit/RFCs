# Sparse observables with extended alphabets

| **Status**        | **Accepted** |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0021                                         |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com), Ian Hincks (ian.hincks@ibm.com) |
| **Submitted**     | 2024-06-19                                   |


## Summary

Add a new Rust-backed observable `SparseObservable` to solve two scaling problems with `SparsePauliOp`.
The new operator will have an extended alphabet of single-qubit terms, in order to avoid exponential explosion in representing projectors, and will be qubit-sparse in that it will not store explicit identities.

This is expected to become part of the `Estimator` interface in the future, but the exact timing and plan for that is out of scope for this document.
We still consider how `SparseObservable` could be used by `Estimator` as part of ensuring the interfaces are suitable.
We separate the ideas of "representation of observables" from "representation of measurement bases", and use different objects to represent and manipulate these.
The former of these is user-facing and will be part of interfaces (including across network boundaries), so needs to be API- and ideally ABI stable.
The latter of these is an implementation detail of `Estimator`s or the circuit-knitting toolbox, so what is used can be changed later if needed for optimisation.


## Motivation

* The current `SparsePauliOp` has exponential memory use when representing terms akin ${\lvert 0\rangle\langle 0\rvert}^{\otimes n}$, making it functionally impossible for a user to ask for expectations of projectors, even though this is easily within our hardware capabilities to calculate.
* The current `SparsePauliOp` uses $\mathcal O\bigl(\text{terms} \times \text{system qubits}\bigr)$ memory, even if the operator is only non-identity on a single qubit, which is a scaling problem as our system sizes increase to thousands of qubits, both in memory usage and in data transfer.
* Changing the natural user-input operator should not impact primitives' ability or efficiency in manipulating the observables to produce hardware-executable programs.


## User Benefit

- End users of all `Estimator` implementations will be able to represent more observables.
- End users of all `Estimator` implementations will use less memory to represent and transmit their observables.
- Primitives implementers will maintain a well-specified serialisable format for data transmission.
- Primitives implementers will have access to high-performance classes for the operations they need.

Note that these benefits can only be realised only once primitives _use_ this new type, which might require some `Estimator` interface evolution out-of-scope of this document.


## Design Proposal

The new `SparseObservable` class is primarily designed to address scaling problems with `SparsePauliOp`.
It will use an extended alphabet to avoid exponential memory explosions in `SparsePauliOp` when representing projection operators (like $`\lvert00\dotsm0\rangle\langle00\dotsm0\rvert`$), and will not store identities on qubits.
Its storage format will be somewhat akin to a CSR/CSC matrix.
Since it is intended to be used in stable interfaces, its internal data representation will be part of its public API, and cannot change once in Qiskit.

In current Qiskit / primitives, the concept of "observable" (`SparsePauliOp`) is described in terms of the object that describes the necessary "measurement bases" (`PauliList`).
With an extended observable alphabet, there can be a many-to-one mapping from observable terms to measurement bases ($Z$ and $\lvert 0\rangle\langle 0\rvert$ in observables both correspond to $Z$ in measurement), so there is a natural break between the two.
The new operator will formalise this within its interfaces; it will have methods to produce the measurement bases needed to measure the terms.

### Definitions

There are several different things that could be "single" or an "atom" here, as there are many groupings.
For clarity, we will use these definitions throughout:

* **abstract observable**:
  One mathematical operator that the user is requesting expectation values for.
  This has the form $\sum_i c_i A_i$ where $`\{c_i\}`$ are complex numbers and $`\{A_i\}`$ are some possibly over-complete spanning set of the operator space.
  Currently, we use `SparsePauliOp` for this, with the representable $`\{A_i\}`$ being the Pauli strings $`\{I,X,Y,Z\}^n`$, represented internally in the Pauli Boolean-symplectic convention.

* **abstract term**:
  A single operator term of the *abstract observable*.
  One of the $`\{A_i\}`$; this is a multi-qubit operator, but has no coefficient that goes with it.
  Currently, we use `Pauli` for this, and `PauliList` is a wrapper type to deal with lists of these.

* **measurement term**:
  A reduction of an *abstract term* to a term that represents the actual measurement that needs to be done to determine it.
  There is a many-to-one mapping of *abstract terms* to *measurement terms*.
  For example, the abstract terms $\lvert 0\rangle\langle 0\rvert$, $\lvert 1\rangle\langle 1\rvert$ and $Z$ are all measured with a projective measurement into the $Z$ basis, and all three can be computed from knowledge of any one of them.

  Currently, we use `Pauli`/`PauliList` for this; our *abstract observables* are already represented in terms of the measurement terms (plus extra explicit identities).
  The measurement terms are not required to be the same size as the abstract terms, even though our current system has this.

### Considered use cases

There are several use-cases for observables throughout the entire life cycle of "a user calls a primitive and gets a result":

1. A user must be able to efficiently construct their operators, which may involve mathematical manipulations as they build up their problem.
   This prioritises efficient construction and manipulation of the objects, while maintaining a clear mathematical meaning in the context of observables.

2. All observables for an estimator pub / batch of pubs must exist simultaneously in memory on the client side and be transmitted over the wire, so must not use excessive memory.

3. An estimator primitive (or circuit-knitting toolbox, etc) must be able to efficiently extract all the terms that correspond to unique measurements that must be made, across the entire broadcast `EstimatorPub`.
   This is not necessarily the same set of mathematical terms as appear in a single operator, and the available groupings are dictated not just by individual mathematical operators or the `ObservablesArray`, but also in how it broadcasts against the `BindingsArray`.

4. After the measurements are made in hardware execution, the values of the observable expectations must be recalculated after.
   This means that the abstract observable should be able to easily and efficiently map back to the measurement terms needed for each of the individual abstract terms.

Cases 1 and 2 are more about the user experience and input format of the primitives.
The obvious difference between the desired alphabet of abstract terms there, and the more restricted set of measurement terms that estimator primitives need to work with internally suggests that case 3 would be better handled by a separate object.
The operations that must be efficient for use-case 3 are also quite different to use-cases 1 and 2; in particular, slicing along qubitwise axes should be efficient for use-case 3, but there is no need to store the term weights or the sum structure in this backend intermediate representation.


### `SparseObservable`: representation of observables

> [!NOTE]
> **Summary**: A new observable class in `quantum_info` will represent Paulis and eigenstate-projectors, and store only non-identity terms.
> This will be for users' benefit, and will be the "natural" input format of `Estimator`.

A new class, `qiskit.quantum_info.SparseObservable`, will (in the future) replace/supplement `SparsePauliOp` as the "abstract observable" input to `Estimator`.
It will first be optimised for minimal memory usage.
Its secondary concern will be to provide efficient mathematical manipulations.

`SparseObservable` will _only_ store single-qubit terms that are not the identity (unlike `SparsePauliOp`).
`SparseObservable` will have an extended alphabet of single-qubit terms:

* $X$, $\lvert +\rangle\langle +\rvert$, $\lvert -\rangle\langle -\rvert$
* $Y$, $\lvert r\rangle\langle r\rvert$, $\lvert l\rangle\langle l\rvert$
* $Z$, $\lvert 0\rangle\langle 0\rvert$, $\lvert 1\rangle\langle 1\rvert$

where the projectors are the associated eigenstates of the Pauli with eigenvalues +1 and -1 respectively.
Each row of this bullet list is a group of compatible measurements; all can be derived from knowledge of one, and the alphabet is chosen to represent what can be efficiently measured by hardware.

For illustrative purposes in the rest of this document, we'll assume that `SparseObservable` will be the "scalar" type of the observable in the `ObservablesArray` in an `EstimatorPub` at some point in the future.
If it shares that role with `SparsePauliOp`, nothing substantial changes about this RFC.
That is the hypothetical Estimator pub would be a three-item record type:

```python
class EstimatorPub:
    circuit: QuantumCircuit
    observables: ND[SparseObservable]
    bindings: ND[Mapping[Parameter, Value]]
```

where `ND` is a stand-in for a type supplying arbitrary-dimensioned n-dimensional broadcasting and slicing, such as a Numpy array of `object`, or a custom container defined in the primitives base classes.
The particulars of `Parameter` and `Value` and the containing mapping are irrelevant to this RFC.

`SparseObservable` will support some set of mathematical operations.
At a minimum, the following will be supported:

* addition of two `SparseObservable`s
* tensor product of two `SparseObservable`s
* evolution of one `SparseObservable` by a Pauli string ($A' = P A P^\dagger$ for `SparseObservable` $A$ and Pauli $P$)
* multiplication by complex scalars
* structural equality of two `SparseObservable`s (structural not mathematical; it's highly inefficient to detect equality if the abstract terms form an over-complete spanning set).

See "Questions" for more on the mathematical-operator support.

Iterator methods for accessing the terms and their coefficients will be supported.
The exact format of these is to be informed by the intended use cases, and there can be more than one iterator method available.


### Representation of measurement bases

> [!NOTE]
> **Summary**: Qiskit will supply some helper functions on `SparseObservable`, but for MVP 1 of the primitive internals, we can keep using `PauliList` for this bit, and we can optimise it in the future if needed.

While the user input is a n-dimensional group of observable sums, the actual execution is serialised and only cares about measurement-basis transformation needed for any given multi-qubit term.
See the "Detailed Design" section for more discussion on exactly how this grouping works.

From Python-space, then, we produce an iterable of "preparation programs" (pairs of `(circuit, bindings)`), and for element of that we have an iterable of `SparseObservable` with we want to map to a group of groups of measurement bases.
The iterable of preparation programs is produced in Python-space by n-dimensional slicing/broadcasting tricks.

A core function of `Estimator` is to group terms that can be measured within the same execution.
There are many ways to do this, and we do not want to tie the observable to one particular implementation.
Qiskit will provide a function `SparseObservable.measurement_bases(*observables)` that takes an arbitrary number of `SparseObservable` instances and returns a set of the measurement bases needed to measure all terms.
A measurement basis is a Pauli string.

Since the number of measurement bases will be (non-strictly) smaller than the total number of terms across all observables, and because only 2 bits of information per qubit is necessary to define the basis, there is not expected to be immediate memory concerns with a representation of this.
Qiskit already has `PauliList` that can serve this purpose; it _could_ be bit-packed to use 8x less memory, but this can be done as a follow-up optimisation if it becomes a bottleneck.
From this point, we can continue to use `PauliList.group_qubitwise_commuting`, or any other future grouping function.

If the output of `measurement_bases` might want to change in the future (such as to a class that implements bit-packing or to a class that does not store qubit identities), we can add an `output` kwarg later.


## Detailed Design

### Data representation of `SparseObservable`

`SparseObservable` will be optimised first for memory usage.
Its internals will be written in Rust.
Its data representation can be thought of as being like:

```rust
// This could also be chosen to be `u64` at the cost of increased memory if
// we decided that 4 billion bits is too restrictive.  (Note: we make the
// `u32` trade-off for bit indices already in Qiskit.)
type BitIndex = u32;

struct SparseObservable {
    // The number of bits the operator acts on.  This is not inferable from
    // any other shape or values, since identities are not stored explicitly.
    num_bits: BitIndex,
    // The coefficients of each abstract term in in the sum.  This has as many
    // elements as terms in the sum.
    coeffs: Vec<Complex64>,
    // A flat list of single-bit (s_) terms.  This is more naturally a
    // list of lists, but is stored flat for memory usage and locality reasons.
    s_terms: Vec<SingleBitTerm>,
    // A flat list of the qubit indices that the corresponding entries in
    // `s_terms` act on.
    s_indices: Vec<BitIndex>,
    // Indices that partition `s_terms` and `s_indices` into sublists for each
    // individual term in the sum.  `boundaries[0]..boundaries[1]` is the range
    // of indices into `s_terms` and `s_indices` that correspond to the first
    // term of the sum.  All unspecified qubit indices are implicitly the
    // identity.  This is one item longer than `coeffs`, since `boundaries[0]` is
    // always an explicit zero (for algorithmic ease).
    boundaries: Vec<usize>,
}
```

The `s_terms`, `s_indices` and `boundaries` entries are somewhat similar to the CSR/CSC sparse-matrix storage format.

`s_terms` _could_ most likely be bit-packed to use up half the space, since most extended alphabets being considered require 4 bits.
The impacts of this on memory are negligible, however:

* `s_indices` is the same length as `s_terms`, and the lengths of these two are the asymptotic scaling limit for memory use.
* the scalar element of `s_indices` is a qubit index, which is likely to be chosen as `u32`, so 4 bytes.
* the scalar element of `s_terms` is 4 bits, which requires 1 byte for alignment if not packed, or 0.5 bytes (average) if so.

Consequently, the savings for bit-packing the `s_terms` vector is asymptotically _at most_ $`0.5 / 5 = 10\%`$, but in practice will be less than this.
Bit-packing will also have some amount of performance impact, and would hinder the creation of cheap "term-view" objects within Rust space, since a term-based slice would not necessarily start on a byte-aligned `SingleBitTerm` values.

The class will be able to present itself without data copying as a Python record type of a similar structure:

```python
class SparseObservableView:
    num_bits: int
    coeffs: Array[complex]
    s_terms: Array[np.uint8]
    s_indices: Array[np.uint32]
    boundaries: Array[np.uintp]
```

where `Array` is a Python object capable of being a memory view onto 1D contiguous C-stored data (not Python objects).
To allow `SparseObservable` to be growable in-place (which in turn allows the `+=` operator to be efficient), this will be a custom Rust-space view that implements the Numpy `__array__` protocol and allows direct reading and writing.[^1]

[^1]: It has to be a custom type that can handle the case of a user holding onto a reference to the view object and then performing an operation that re-allocates the Rust-space buffers (like a grow-in-place that exceeds the available capacity).

This view form will be available to enable serialisation from Python space.

A enumeration `SingleBitTerm` will be available in Python space that provides named access to the integer values for each item of `s_terms`.

### Choice of abstract-term extended alphabet

`SingleBitTerm` is an `enum` of the extended alphabet, where each term has a well-defined integer representation.
A `SparsePauliOp` would use $`\{I,X,Y,Z\}`$ here, which is a basis of the single-qubit space.
`SparseObservable` will use a larger alphabet of terms here, which is an over-complete spanning set, so operators like ${\lvert 0\rangle\langle 0\rvert}^{\otimes n}$ are representable with $\mathcal O(n)$ memory requirements instead of `SparsePauliOp`'s $\mathcal O(2^n)$.

Since this will be an input format to the primitives, the extended alphabet should be exactly fixed, so primitive implementers can rely on it.
See the "Alternative Approaches" below for some exploration of what it would mean to have this chosen dynamically.

The alphabet should be designed specifically so that an observable that can be _represented_ efficiently can also be _executed_ efficiently.
Allowing items such as $\lvert 0\rangle\langle 1\rvert$ into the alphabet could result in an operator that needs linear space to be stored, but exponential numbers of executions to evaluate.
As best as possible, all observables that can be executed efficiently should also be representable efficiently; `SparsePauliOp` fails on this front.


### Construction of `SparseObservable`

We can supply several constructors and converters into `SparseObservable`.

#### High level, user facing

At the high level for users, it is easy to supply a `from_sparse_pauli_op` method.
`SparsePauliOp.from_list` and `SparsePauliOp.from_sparse_list` both have natural extensions to `SparseObservable`; simply extend the alphabet used in them.
`from_sparse_list` is the most natural for `SparseObservable`.

Most `quantum_info` objects heavily overload their default constructor to do significant type inference, and perform many types of move-semantics, copy-semantics and conversion constructors.
To keep the "feel" of the module similar, it is likely best to continue with this.
The default constructor will accept the following objects:

* A single string, treated as a dense Pauli-and-projectors string and lifted to a single observable with 1.0 coefficient.
* A `Pauli` or `SparsePauliOp` object, converted directly.
* Another `SparseObservable`, which will evoke copy semantics.

For example:

```python
from qiskit.quantum_info import SparseObservable, Pauli, SparsePauliOp

pauli = Pauli("XYZ")
spo = SparsePauliOp.from_list([("ZIIIX", 1.0), ("IIIIX", -1.0)])

# Construct a `SparseObservable` from a single extended-alphabet string.
SparseObservable("XIIZ0+")

# Construct a `SparseObservable` from a `Pauli` using the default constructor,
# which represents the same operator.  Also available as
# `SparseObservable.from_pauli`.
SparseObservable(pauli)

# Construct a `SparseObservable` from a `SparsePauliOp` using the default
# constructor, again.  This operator _could_ be more efficiently represented in
# `SparseObservable` using the |1><1| projector, but finding the most efficient
# factorisation is (in general) a very hard problem, so we don't attempt it.
# Also available as `SparseObservable.from_sparse_pauli_op`.
SparseObservable(spo)

# Use the `SparsePauliOp`-like fully sparse list format to construct an
# operator.  This also allows using the extended alphabet (here, the Z-basis
# eigenstate projectors).  This is will likely be the fastest choice for
# users of the high-level interface.
SparseObservable.from_sparse_list([
    ("000", (12, 11, 10), 1.0),
    ("111", (100, 99, 98), 1.0),
], num_qubits=200)
```


#### Low level, libraries and deserialisation

To support low-level fast-path construction, intended for use by wrapper libraries, Qiskit itself, and the highest-performance code, `SparseObservable` will also have an `uninitialized` method to allocate the buffers from Rust space, then expose them as uninitialised Numpy arrays.
There will also be a safer, copy-in constructor.
These methods are not intended to have significant use by users, but are part of providing efficient serialisation/deserialisation and Python-space mathematical operations for libraries:

```python
import numpy as np
from qiskit.quantum_info import SparseObservable

# Using the copy-in constructor to make the Z_0 + Z_1 + Z_2 + ... operator
# by manually providing the data buffers.
num_bits = 100
terms = np.full((num_bits,), SparseObservable.Term["Z"], dtype=np.uint8)
indices = np.arange(num_bits, dtype=np.uint32)
coeffs = np.ones((num_bits,), dtype=complex)
boundaries = np.arange(num_bits, dtype=np.uintp)
# This copies out of the buffers, because we need the base allocations to be
# owned by Rust space.  For performance, this will not check the values of the
# input arrays for validity.
SparseObservable.from_raw_parts(num_bits, coeffs, terms, indices, boundaries)
```

### Procedure for execution from an `EstimatorPub`

> [!NOTE]
> The purpose of this section is mostly an exercise in checking that the pathways aren't constrained by the new types.
> I haven't written full detail of the n-dimensional slicing, just indicated where it's happening, where it's temporarily flattened, and where it's reshaped back.

An `EstimatorPub` contains an ND container of parameter bindings, an ND container of `SparseObservables`, and the state-preparation circuit.
The bindings and observables are broadcast against each other, producing the shape of the output.

For efficient execution of the primitive circuits on hardware, the final shape is not especially important, nor are the exact abstract terms used in the observable.
What matters is taking the entire broadcast, and grouping terms that can be measured within the same hardware execution.
The components of this are:

1. Within a `SparseObservable`, all terms are eligible for hardware-execution grouping.
2. For broadcast axes where the "bindings" length is 1, the terms of all observables along that axis are eligible for hardware-execution grouping.
3. Expectation values of any single-qubit Pauli and its associated eigenstate projectors can all be evaluated by the same execution.
4. Any group of qubitwise-commuting operator strings can be evaluated with the same execution.

An `Estimator` evaluates the conditions of points (1) and (2) to find a groups of observables that are eligible for execution-wise grouping.
This is done by Python-space n-dimensional broadcasting of the `observables` array and then flattening the contained `SparseObservables` into only measurement terms.

Next, we use points (3) and (4) to produce a set of groups of compatible measurements for each group of observables.
We store a hash map of measurement bases to the execution index that was used for that basis.
The outer groups are still n-dimensional, but the inner groups no longer need to be stored as such; each element of the inner group refers to one shot loop (up to twirling, etc) of distinct executions on the QPU, so the inner group is naturally serial.

Each preparation program is then run with each of the measurement bases necessary (plus error-mitigation overhead, etc).
The return is the estimation of the expectation value measurement-basis Pauli string, from which the associated eigenstate-projection operators can be constant-time derived.
For each preparation program for each `SparseObservable` (respecting n-dimensional broadcasting), the observable can be evaluated:

1. For each term of the `SparseObservable`, find the equivalent measurement term.
2. Use the hash map of measurement bases to execution indices to look up the error-mitigated counts for that term.
3. Use the original abstract term to compute the expectation value from the counts.
4. Weighted-sum all the values for each abstract term with its coefficient.

This procedure is repeated for each observable in the n-d broadcast.
This then produces the n-dimensional `values` and `stds` fields of the `PubResult`.

In general, `qiskit.primitives` can expose Rust-space helper functions from `qiskit._accelerate` if any of this evaluation is a performance bottleneck in practice.

### Memory comparisons

For simplicity, I'm just talking about 64-bit systems here using 32-bit integers to represent bit indices (where appropriate).

Let us assume that the number of non-identity single-qubit entries in any given term of an observable is, on average, a constant $k$ that depends on the `Estimator` problem.
$k$ is non-strictly less than the number of qubits.

`SparseObservable` uses 104 bytes inline, plus heap allocations of $\bigl((24 + 5k)\times\text{terms} + 8\bigr)$ bytes.  (The factor of $5k$ can be reduced to $4.5k$ with bit-packing of the alphabet terms.)

A hypothetical Rust-space `SparsePauliOp` that uses maximal bit-packing of the symplectic arrays and merges the phases into the complex double-precision coefficients uses around 80 bytes inline, plus heap allocations of $\text{terms}\times(\text{qubits}/4 + 16)$ bytes.

The existing Python-space `SparsePauliOp` uses rather more constant-offset storage, plus heap allocations of $\text{terms}\times(2\times\text{qubits} + 17)$ bytes.

Ignoring the exponential explosion (which `SparseObservable` is _always_ better at than `SparsePauliOp`), `SparseObservable` is better/equal to the theoretical best `SparsePauliOp` if the filling factor $k \le \text{qubits}/20$ (for the current `SparsePauliOp`, it's $k \le 0.4\times\text{qubits}$).
For a current 133q system, that condition is that on average, observable terms have 6-7 non-identities in them.
For a 2000q system, that condition is 100 non-identities per term.

Some comparisons (in bytes):

| Observable | `SparseObservable` | Hypothetical best `SparsePauliOp` |
|:-----------|:-------------------|:----------------------------------|
| Weighted sum of $n$  single-qubit $Z$ operators | $29n + 8$ | $\frac14 n^2 + 8n$ |
| Weighted sum of all pairwise $ZZ$ operators for $n$ qubits | $17n^2 - 17n + 8$ | $\frac18n^3 + \frac{63}{8}n^2 - 8n$ |
| Projector onto all-zeros state of $n$ qubits | $5n + 32$ | $2^n (\frac14 n + 16)$ |
| Single random Pauli string on $n$ qubits | $\frac{15}{4}n + 32$ | $\frac14 n + 16$ |

The takeaways here:

* for operators with small numbers of non-identity qubits, `SparseObservable` scales polynomially better.
* for operators involving projectors, `SparseObservable` scales exponentially better.
* for qubit-dense observables, `SparseObservable` scales constant-size worse by a factor of 15-20.

## Alternative Approaches

### Use `SparsePauliOp`, but with a separate `qargs` tuple to subset the qubits

`QuantumCircuit` treats implicit identities by having each full `CircuitInstruction` be a record of the gate applied, and the qubits it is applied to.
All unnamed qubits implicitly undergo the identity.
The `Estimator` observables could add a similar field at the level of the complete observable.

`SparseObservable` uses a form of this, except the qubit subsetting is done at the level of individual _terms_.
Consider the operator $\sum_i Z_i$, where $Z_i$ is a Pauli $Z$ on qubit $i$.
`SparsePauliOp` would need $\mathcal O(n^2)$ memory to store the operator, for $n$ qubits involved in the operator, as each "row" in the symplectic Pauli representation stores explicit identities for all qubits in the subset.
This is potentially less than the full device width, but in contrast, `SparseObservable` needs $\mathcal O(n)$ memory for the same object.

Fundamentally, while subsetting qubits at the level of the full operator could have helped somewhat during the switch to ISA circuits in the primitives, it doesn't address the general scaling to 1000+q operators made up of single-qubit terms.
Continuing to use `SparsePauliOp` also makes it impossible to represent projection-like operators, even on existing real-world systems, because of the exponential memory scaling.


### Allow the `SparseObservable` extended alphabet to be extended / configured

From Rust space, most construction and mathematical-manipulation related operations do not require the particular extended alphabet to be known at compile time, provided a few invariants are upheld:

* There is a compile-time known upper bound on the number of single-qubit terms in any given alphabet (determines the storage requirements).
* The abstract operator spanning set is made up of single-qubit separable operators.
* The type constructor of the genericised `SparseObservable` can supply look-up tables for various implemented mathematical methods.

In practice, however, this type will be part of the primitives interface.
Primitives _could_ be made generic over the alphabet, but it's unclear what benefit that would really have; this object is about abstract mathematical representations, not about internal details of any one primitive implementer.

`Estimator` implementers definitely care about some particular specifics, though, including the mapping of the abstract terms to measurement terms, and the subsequent basis transformation that needs to be applied to a circuit to map each measurement term onto a $Z$-basis measurement.
Making the alphabet customisable makes it harder to do this, and ties our hands in the mathematical operations we can add to the operator.


### Make `SparseObservable` a natural n-dimensional object

The potential ask here is to join the functionality of the ND broadcasting with the underlying object, since `SparseObservable` will appear in the primitives in ND containers.

Much of the point of Numpy-like ND slicing in Python space is because the underlying structure is contiguous, and it's possible to slice along "qubit" columns etc in `SparsePauliOp` to create view objects in constant time, still allowing efficient Numpy operations on the objects.
This isn't possible in the qubit-sparse `SparseObservable`; assuming the `s_indices` are stored term-wise sorted, finding the single-qubit term in a complete term is $\mathcal O\bigl(\lg(\text{explicitly stored qubits})\bigr)$.
Further, creating an intermediate "slice" object typically requires either data copying, or a complex range of broadcasting combinators (new Rust-space classes), making its efficiency far less than for an operator with predictable strides.

Lastly, one "axis" of even the `SparseObservable.coeffs` (which don't have the "packed list-of-ragged-lists" behaviour addressed above) shape would be special, because it comes with implicit sum semantics along the axis.
Adding the n-dimensionality *within* `SparseObservable` would be putting the slicing operations in the wrong place, and meaning that care was always particularly needed to avoid slicing an axis that shouldn't be possible to slice.
All this can be avoided by using `SparseObservable` purely as the scalar object _within_ some other container object providing the ND functionality.

The remaining uses of ND slicing are expected to be more useful in the creation and manipulation of the representation of groups of *measurement terms* rather than the *abstract observables* that `SparseObservable` represents.
These cases are handled elsewhere in this document.


### Instead of using an extended alphabet and separate coefficients, store explicit 1q matrices for each term

Extending the alphabet of `SparseObservable` from beyond the Paulis means that certain binary mathematical operations that were upper-bounded at a quadratic number of terms for `SparsePauliOp` are no longer so bounded.
(See the "Questions" section for greater discussion of this.)
For example: matrix multiplication of two operators of the form $\sum_i c_i A_i$ for an operator alphabet $`\{A_i\}`$ containing the Pauli operators and projectors onto their eigenstates is not closed in the same way; a projector multiplied by a different Pauli produces a term like $\lvert 0\rangle\langle 1\rvert$, which requires the sum of two alphabet terms to express.
This leads to the same exponential explosion problems as `SparsePauliOp` has in representing projectors.
This happens because the alphabet is formed of Hermitian operators (even though the complete observable need not be), but matrix multiplication does not preserve hermicity.

This can in theory be avoided by storing an explicit $2\times2$ matrix of coefficients for single-qubit term.
For observables whose terms all involve only one or two qubits, the memory impacts are potentially acceptable, but the constant-factor scaling as the number of non-identity qubits increases is far poorer overall.
Asymptotically, doing this would have a memory usage of 36 bytes per non-identity qubit + 8 bytes per term, compared to 5 bytes per non-identity qubit + 16 bytes per term, and would correspondingly be more expensive to compute with and to decompose to "measurement" operations.

Supporting things like this efficiently in the observable is also perhaps misleading for the user; the _measurement terms_ needed to support things like $\lvert 0\rangle\langle 1\rvert$ still involve the exponential explosion (they go like $X + iY$), which could easily lead to a user getting needing / being charged for exponentially scaling runtime from an operator that appeared benign.


## Questions

### Should `SparseObservable` allow parametric observables?

The definition given of `SparseObservables` limits the coefficients to be double-precision complex floats.
`SparsePauliOp` technically allows observables to be parametrised in terms of Qiskit's symbolic `Parameter` class (or really any Python object).

Should `SparseObservable` also support these uses?

Pros of support:

* More flexibility in what can be represented in the operators.

Cons:

* Operations on the type will be more complex, and the general case will need to hold the Python GIL in Rust space to act on the observables.
* More complex code is needed from Rust space to handle mathematical manipulations efficiently in the happy path of "no Python-space parametrisation".

Jake: personally I'd avoid this unless we have a really strong compelling use-case for giving it first-class support.
A user can always work around this simply by splitting the terms of their sum into different broadcast axes in the estimator pub, then calculating the sums themselves, which gives them far more freedom.


### What mathematical operations should `SparseObservable` support?

From above, the minimum set is:

* addition of two `SparseObservable`s
* tensor product of two `SparseObservable`s
* evolution of one `SparseObservable` by a single term
* multiplication by complex scalars
* structural equality of two `SparseObservable`s (structural not mathematical; it's highly inefficient to detect equality if the abstract terms form an overcomplete spanning set).

There are several others that might be possible too.
These operations could all fairly easily be supported:

* Evolution of `SparseObservable` by another: this is completely doable, just naturally has quadratic complexity.
* Tidy-up structural compaction of the operator: summing all terms that share the same abstract operator, removing zeros at some specified tolerance.

These notable operations have mathematical problems with their support:

* Composition (matrix multiplication) of one `SparseObservable` with another.
  This can naturally result in terms that are outside the extended alphabet: consider $\lvert 0\rangle\langle 0\rvert \cdot X = \lvert 0\rangle\langle 1\rvert$.
  If this is supported, there are either further restrictions on the extended alphabet (making it massively wider) in order to keep the group closed under this operation, or matrix-multiplication can easily result in the same exponential explosion of memory requirements that this operator was designed to solve.

## Future Extensions

### Custom measurement-term collection type

This RFC suggests continuing to use `PauliList` or at least the underlying symplectic representation for the measurement-term grouping for the near future.

Currently, this object is primarily an internal implementation detail of how IBM primitives split up their inputs for efficient hardware execution, and is not required to be part of the primitives interface, so changing it when required is much simpler.
If the dense-memory concerns begin to be more of a problem within actual usage of the measurement-term grouping, we can revisit making new objects to handle these calculations more efficiently.

In the immediate term, the 100--500q scale should not cause excessive memory usage, even with the byte-aligned symplectic representation of `PauliList`.


### Extended-alphabet version of a qubit-dense operator

There are still cases where `SparsePauliOp` is significantly more efficient than `SparseObservable`, notably when the abstract terms are all non-sparse Pauli strings.
If the case of non-sparse projection operators is also something that should be made efficient, we may need to later add another object.
Such an object will not have asymptotic scaling better than `SparseObservable`, but its constant factors can be significantly better, and it would be faster to manipulate.
