# Base Primitive and Units of Primitive Work

| **Status**        | **Accepted**             |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0017                                         |
| **Authors**       | Lev Bishop (lsbishop@us.ibm.com)             |
|                   | Ian Hincks (ian.hincks@ibm.com)              |
|                   | Blake Johnson (blake.johnson@ibm.com)        |
|                   | Chris Wood (cjwood@us.ibm.com)               |
| **Submitted**     | 2023-10-26                                   |
| **Updated**       | 2023-12-20                                   |

## Summary

The primitives form an execution framework for QPUs and their simulators.
Each primitive specializes at some computational task.
For example, the `Estimator` specializes at computing expectation values of user-provider observables.

Different types of primitives are implemented as different abstract subclasses of `BasePrimitive` (e.g. `BaseEstimator`), and different primitive implementations as further subclasses (e.g. `qiskit_aer.primitives.Estimator`).
This RFC proposes the structure and format of various container types that can be used by base primitives to store input and output values.

## Motivation

The central motivation of this RFC is to more clearly emphasize that primitives are an execution framework on top of which applications (not just algorithms) can be built. 
We aim to help each type of primitive to clearly state what units of quantum work it is able to perform.
We do this by proposing a set of containers that various `run()` methods can use (possibly subclassed) as input and output types.
In doing so, we are also proposing a set of names that can help us standardize the way we talk about execution in documentation, and throughout the stack.

## User Benefit

We expect that users will generally benefit from a more homogenous experience across the primitives, in terms of how they would submit jobs, and in how they would collect the data out of their results.
These containers would also make it possible for base primitives to have multiple, named output types.
This would enable, for example, primitives to have separate output fields for each output variable of a particular circuit, or to take standard error/confidence intervals out of the metadata and put it right alongside the first moment data.

## Design Proposal

We propose to encourage base primitives, like `BaseSampler` and `BaseEstimator`, to have a `run` method typed as

```python
class BaseFoo:
    def run(self, pubs: FooPubLike | Iterable[FooPubLike]) -> Job[PrimitiveResult[FooPubResult]]:
        """Run one or more input PUB (Primitives Unified Bloc) and return a result for each one."""
```

where `FooPub` is equal to or derives from `Pub`, `FooPubLike` is a union type that is easily coercible into a `FooPub` via the static method `FooPub.coerce`, and where `FooPubResult` derives from `PubResult`.

The base containers `PrimitiveResult,`, `Pub`, and `PubResult` are described in the next sections.
`Job[T]` is any `qiskit.provider.JobV1` whose result method returns type `T`.

Any primitive following the above pattern could be used as follows:

```python
# instantiate the primitive
foo = Foo()

# run the primitive with three PUBs
job = foo.run([pub0, pub1, pub2])

# block for results
result = job.result()

# get data from second PUB
result1 = result[1]

# get particular data from this result (the available fields depend on the primitive type and PUB,
# and this example is not proposing any specific names)
alpha_data = result1.data.alpha
beta_data = result1.data.beta
expectation_values = result1.data.expectation_values
```

## Detailed Design

### PUBs

We propose the concept of a _PUB (Primitive Unified Blocs)_, which we define as _a single circuit along with auxiliary data required to execute the circuit relative to the primitive in question_. This concept is general enough that it can be used for all primitive types, current and future, where we stress that what the “auxiliary data” is can vary between primitive types. 

```python
# prototype implementation for the RFC
@dataclass(frozen=True)
class Pub:
    circuit: QuantumCircuit
```

Different primitive types (such as `Estimator`) are intended to subclass this class, adding auxiliary fields as 
required.

### DataBin

A data bin is a namespace for storing data.
The fields in the namespace, in general, depend on the PUB that was executed (not the `type` of the PUB).
For example, a `Sampler` will have fields for each output (as defined by the OpenQASM 3 spec, but in Qiskit, you can currently take "output" to mean the names of the classical registers) of the PUB's circuit.
The value of each field will store the corresponding data.

All primitives will store their data in `DataBin`s.
There will not be subclassing to the effect of `SamplerDataBin < DataBin`; look instead to `PubResult` for such needs.

```python
# Make a new DataBin class. make_data_bin() is analagous to dataclasses.make_dataclass().
# The shape is an optional argument which indicates that all values are to share the same leading shape.
# To put the following generic example into context, if it helps, imagine that this code lives in the 
# BaseSampler implementation, and a data bin class is being created to store the results from 
# particular SamplerPub instance whose circuit has two output registers named alpha and beta, and 
# that the PUB itself has shape (5, 4).
data_bin_cls = make_data_bin({"alpha": NDArray[np.float], "beta": NDArray[np.uint8]}, shape=(5, 4))

# make an instance with particular data
data_bin = data_bin_cls(
    alpha=np.empty((5, 4, 1024), dtype=np.float), 
    beta=np.empty((5, 4, 1024, 127))
)

# access items as attributes
alpha_data = data_bin.alpha

# access items with subscripts
alpha_data = data_bin["alpha"]
```

### PubResult

A `PubResult` is the result of running a single `Pub` and does three things:

  * Stores a `DataBin` instance containing the data from execution.
  * Stores the metadata that is possibly implementation-specific, and always specific to the executed PUB.
  * (subclasses) Contains methods to help transform the data into standard formats, including migration helpers.

We generally expect each primitive type to define its own subclass of `PubResult` to accomodate the third item, though `PubResult` has no methods that need to be abstract.

We elect to have a special container for the data (`DataBin`) so as not to pollute the `PubResult` namespace with pub-specific names, and to keep data quite distinct from metadata.

```python
# return a DataBin
pub_result.data

# return a metadata dictionary
pub_result.metadata
```


### PrimitiveResult

`PrimitiveResult` is the type returned by `Job.result()` and is primarily a `Sequence[PubResult]`, with one entry for each `Pub` input into the primitive's run method.

Secondarily, it has a metadata field for those metadata which are not specific to any single PUB.
