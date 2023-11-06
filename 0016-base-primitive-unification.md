# Base Primitive and Units of Primitive Work

| **Status**        | **Proposed/Accepted/Deprecated**             |
|:------------------|:---------------------------------------------|
| **RFC #**         | 0016                                         |
| **Authors**       | Lev Bishop (lsbishop@us.ibm.com)             |
|                   | Ian Hincks (ian.hincks@ibm.com)              |
|                   | Blake Johnson (blake.johnson@ibm.com)        |
|                   | Chris Wood (cjwood@us.ibm.com)               |
| **Submitted**     | 2023-10-26                                   |
| **Updated**       | YYYY-MM-DD     

## Summary

The primitives form an execution framework for QPUs and their simulators.
Each primitive specializes at some computational task.
For example, the `Estimator` specializes at computing expectation values of user-provider observables.

Different types of primitives are implemented as different abstract subclasses of `BasePrimitive` (e.g. `BaseEstimator`), and different primitive implementations as further subclasses (e.g. `qiskit_aer.primitives.Estimator`).
This RFC proposes the structure and format of various container types that can be used by base primitives to store input and output values.

## Motivation

The central motivation of this RFC is to more clearly emphasize that primitives are an execution framework on top of which applications (not just algorithms) can be built. 
We aim to help each type of primitive to clearly state what units of quantum work it is able to perform.
We do this by proposing a set of containers that various `run()` can use (possibly subclassed) as input and output types.
In doing so, we are also proposing a set of names that can help us standardize the way we talk about execution in documentation, and throughout the stack.

## User Benefit

We expect that users will generally benefit from a more homogenous experience across the primitives, in terms of how they would submit jobs, and in how they would collect the data out of their results.
These containers would also make it possible for base primitives to have multiple, named output types.
This would enable, for example, primitives to have separate output fields for each output variable of a particular circuit, or to take standard error/confidence intervals out of the metadata and put it right alongside the first moment data.

## Design Proposal

We propose to encourage base primitives, like `BaseSampler` and `BaseEstimator`, to have a `run` method typed as

```python
class BaseFoo:
    def run(self, tasks: FooTaskLike | Iterable[FooTaskLike]) -> Job[PrimitiveResult[FooTaskResult]]:
        """Run one or more input tasks and return a result for each one."""
```

where `FooTask` is equal to or derives from `Task`, `FooTaskLike` is a union type that is easily coercible into a `FooTask` via the static method `FooTask.coerce`, and where `FooTaskResult` derives from `TaskResult`.

The base containers `PrimitiveResult,`, `Task`, and `TaskResult` are described in the next sections.
`Job[T]` is any `qiskit.provider.JobV1` whose result method returns type `T`.

Any primitive following the above pattern could be used as follows:

```python
# instantiate the primitive
foo = Foo()

# run the primitive with three tasks
job = foo.run([task0, task1, task2])

# block for results
result = job.result()

# get data from second task
result1 = result[1]

# get particular data from this result (the available fields depend on the primitive type and task,
# and this example is not proposing any specific names)
alpha_data = result1.data.alpha
beta_data = result1.data.beta
expectation_values = result1.data.expectation_values
```

## Detailed Design

### Task

We propose the concept of a _task_, which we define as _a single circuit along with auxiliary data required to execute the circuit relative to the primitive in question_. This concept is general enough that it can be used for all primitive types, current and future, where we stress that what the “auxiliary data” is can vary between primitive types. 

```python
# prototype implementation for the RFC
@dataclass(frozen=True)
class Task:
    circuit: QuantumCircuit
```

Different primitive types (such as `Estimator`) are intended to subclass this class, adding auxiliary fields as 
required.

### DataBin

A data bin is a namespace for storing data.
The fields in the namespace, in general, depend on the task that was executed (not the `type` of the task).
For example, a `Sampler` will have fields for each output (as defined by the OpenQASM 3 spec, but in Qiskit, you can currently take "output" to mean the names of the classical registers) of the task's circuit.
The value of each field will store the corresponding data.

All primitives will store their data in `DataBin`s.
There will not be subclassing to the effect of `SamplerDataBin < DataBin`; look instead to `TaskResult` for such needs.

```python
# Make a new DataBin class. make_data_bin() is analagous to dataclasses.make_dataclass().
# The shape is an optional argument which indicates that all values are to share the same leading shape.
# To put the following generic example into context, if it helps, imagine that this code lives in the 
# BaseSampler implementation, and a data bin class is being created to store the results from 
# particular SamplerTask instance whose circuit has two output registers named alpha and beta, and 
# that the task itself has shape (5, 4).
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

### TaskResult

A `TaskResult` is the result of running a single `Task` and does three things:

  * Stores a `DataBin` instance containing the data from execution.
  * Stores the metadata that is possibly implementation-specific, and always specific to the executed task.
  * (subclasses) Contains methods to help transform the data into standard formats, including migration helpers.

We generally expect each primitive type to define its own subclass of `TaskResult` to accomodate the third item, though `TaskResult` has no methods that need to be abstract.

We elect to have a special container for the data (`DataBin`) so as not to pollute the `TaskResult` namespace with task-specific names, and to keep data quite distinct from metadata.

```python
# return a DataBin
task_result.data

# return a metadata dictionary
task_result.metadata
```


### PrimitiveResult

`PrimitiveResult` is the type returned by `Job.result()` and is primarily a `Sequence[TaskResult]`, with one entry for each `Task` input into the primitive's run method.

Secondarily, it has a metadata field for those metadata which are not specific to any single task.