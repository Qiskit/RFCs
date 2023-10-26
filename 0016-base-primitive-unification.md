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
Because the constructor of these subclasses is reserved for implementation-specific setup, the main entry-point to execution is the `.run()` method of a particular class.
However, as it stands, `BasePrimitive` does not provide `.run()` as an abstraction; each type of primitive gets to define its own signature.

This RFC proposes an abstraction for `BasePrimitive.run()` as well as structures for its inputs and outputs.

## Motivation

The central motivation of this RFC is to more clearly emphasize that primitives are a well-defined execution framework on top of which applications can be built.
We aim to force each category of primitive to clearly state what units of quantum work (tasks) it is able perform, and what outputs can be expected.
It is understood that having the proposed abstract base method alone will not necessarily improve the day-to-day life of a typical user of the primitives, and neither will it necessarily improve the ease of implementing new primitive types or implementations: this change is mainly about clarifying the shared nature of the primitives, and reflecting this view in abstractions.

## User Benefit

One class of benefited users are application frameworks that are built on top of the primitives, which, following this proposal, will be able to exploit the promise of a homogenous experience across all primitives.
As an example, `qiskit_experiments` defines a suite of common diagnostic, calibration, and characterization experiments.
Currently, it is designed around the `Backend.run` execution interface.
One could imagine that, for example, instead of each `Experiment` providing a `circuits` attribute specifying the circuits to run through `Backend.run`, they could rather each specify a `tasks` attribute indicating which primitive units of work need performed.
The `qiskit_experiments` execution and analysis machinery could then rely on the abstraction of `BasePrimitive.run` when reasoning about dispatching and collecting results.

## Design Proposal

We propose the following abstraction, where descriptions of all types can be found in the detailed design section.

```python
In = TypeVar("In", base=Task)
Out = TypeVar("Out", base=TaskResult)
Shape = Tuple[int, ...]

class BasePrimitive(ABC, Generic[In, Out]):
    @abstractmethod
    def run(self, tasks: In | Iterable[In]) -> Job[PrimitiveResult[Out]]:
        """Run one or more input tasks and return a result for each one."""

    @classmethod
    def result_type(cls, task: In) -> DataBundle:
        """Return the typed data namespace that can be expected if the given 
        task were to be run."""
        # concrete implementation in terms of cls._output_fields(task)

    @classmethod
    @abstractmethod
    def _output_fields(cls, task: In) -> Iterable[Tuple[str, type, Shape]]:
        """Prescription of field names, types, and shapes for the task."""
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

Different primitive types (such as `Estimator`) are intented to subclass this type

### DataBundle

A data bundle is a namespace for storing data.
The fields in the namespace, in general, depend on the task that was executed.
For example, a `Sampler` will have fields for each output (as defined by the OpenQASM 3 spec, but in Qiskit, you can currently take "output" to mean the names of the classical registers) of the task's circuit.
The value of each field will store the corresponding data.
The names, types, and shapes of each field in the namespace can be queried before execution via the abstract class method `BasePrimitive.result_type(Task) -> DataBundle`.

All primitives will store their data in `DataBundle`s.
There will not be subclassing to the effect of `SamplerDataBundle < DataBundle`; look instead to `TaskResult` for such needs.

### TaskResult

A `TaskResult` is the result of running a single `Task` and does three things:

  #. Stores a `DataBundle` instance bcontaining the data from execution.
  #. Stores the metadata that is possibly implementation-specific, and always specific to the executed task.
  #. (subclasses) Contains methods to help transform the data into standard formats, including migration helpers.

We generally expect each primitive type to define its own subclass of `TaskResult` to accomodate the third item, though `TaskResult` has no methods that need to be abstract.

We elect to have a special container for the data (`DataBundle`) so as not to pollute the `TaskResult` namespace with task-specific names, to enable a clean `BasePrimitive.result_type` formalism, and to keep data quite distinct from metadata.

### PrimitiveResult

`PrimitiveResult` is the type returned by `Job.result()` and is primarily a `Sequence[TaskResult]`, with one entry for each `Task` input into the primitive's run method.

Secondarily, it has a metadata field for those metadata which are not specific to any single task.

## Alternative Approaches

The alternate approach is what currently exists: to not have a common abstraction.

## Future Extensions

The future extensions are primarily in the choices that particular primitive types would make to satisfy this proposal.