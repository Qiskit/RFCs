# [WIP] Aqua's QuantumAlgorithm: Interfaces, Instance Reuse, and Organization

| **Status**    | **Proposed**                        |
| :------------ | :---------------------------------- |
| **RFC #**     | ####                                |
| **Authors**   | Julien Gacon (jul@zurich.ibm.com),  |
|               | Stefan Wörner (wor@zurich.ibm.com), |
|               | Donny Greenberg (donny@ibm.com)     |
| **Submitted** | 2020-01-28                          |
| **Updated**   | YYYY-MM-DD                          |

## Summary

Aqua's quantum algorithms are of type `QuantumAlgorithm`, which dictates that all algorithms must be executed via a  `run` method whose arguments specify the backend used for circuit execution. This structure makes passing algorithm instances to meta-algorithms difficult and hacky as once instantiated the problem the algorithm solves is essentially fixed. This RFC discusses the issues of the current `QuantumAlgorithm` and tries to assess, whether this base type is still useful or if it might complicate the development of quantum algorithms.

The below proposal introduces the following:
1. **Interface Classes** - Non-mutually-exclusive base classes for various types of algorithms, such as MinEigenSolver, ClassicalAlgorithm, Classifier, or AmplitudeEstimator, enforcing a consistent execution signature (`get_min_eigenvalue()`) which is still flexible enough to support diverse execution flows for different algorithm _types._
1. **Remove `.run` Requirement** - For algorithms where a `.run(backend)` execution interface is not natural, such as Classifiers (init, train, test, predict), remove the requirement that QuantumAlgorithms include a `run(backend)` method as it is no longer required for dictionary-based execution.
1. **Incomplete Construction and Reuse** - Give algorithms the discretion for some constructor parameters to be left empty for population later, allowing sub-algorithms to be passed around without having to be fixed to a specific problem, and for some execution methods to be reusable with components passed in (e.g. `my_expectation.get_expectation(my_circ)`).
1. **Directory Simplification** - Remove the non-standard `single-sample`, `many-sample`, and `adaptive` nomenclature from the algorithms directory in favor of a flatter, simpler structure (TODO link below), and promote the Optimizers into the algorithms directory, as they are algorithms.

## Motivation
- Why are we doing this?
    - Current structure:
        - To run a quantum algorithm, one initializes the algorithm with all necessary properties
        and then calls `run(backend)`, specifying a backend (or quantum instance) for the
        circuit execution. This method returns the algorithm result.
        - All quantum algorithms inherit from the abstract `QuantumAlgorithm` type, which requires
        an implementation of the actual algorithm execution via the `_run()` method.
        This method is called, when the user calls `run(backend)`.
        - Noteably,
          - Not all algorithms construct circuits or require a quantum backend
          - The problem the algorithm solves is defined in the initializer

        The execution method is only allowed to take a `QuantumInstance` plus configuration. Thus, all configuration must take place before the execution, mostly in the initializer. In this execution, most (all?) algorithms construct circuits using the information provided in the initializer, execute all iterations, process the outcome and return the result.
    - Problems introduced by the legacy structure:
        1. Algorithms are fixed, but the problems they solve are not. Why do we have to specify
        a new algorithm instance for each new problem when we could just input new problem information when
        we execute?
        1. Algorithms are *too* interchangeable when they all share the same `.run` method and base class. It is currently very difficult for a meta-algorithm to validate whether an algorithm passed to it actually does the activity it needs. This reduces the interchangeability of the algorithms. It would be better for an algorithm to simply check the algorithm type and access the information it needs via an enforced abstractmethod.   
        1. Missing flexibility: configuring the algorithm in the initializer makes the algorithm instance problem-specific. We are therefore unable to pass an algorithm instance that can solve different problems to meta algorithms. Examples: ADMM, any optimization, etc.
        1. The term "algorithm" is extremely broad. Classical algorithms don't share a common base
        class (compare Qiskit, scikit, etc.). Why should quantum algorithms share one?
        1. Naming: every single quantum algorithm is executed using `run`, though other names
        can be much more expressive. Optimizers `optimize` or `minimize` a quantity, ML models `train` or `test` or `predict`, HHL `solve`s, etc. Classical algorithms do not have this naming restriction,
        why should quantum algorithms have them?
        1. If an algorithm runs and compares multiple backends or settings, this is not supported in
        a convenient manner.
        1. Algorithms are almost always executed in hardware-specific ways, and only doing most of the execution setup (e.g. constructing circuits) at the very last momemnt in `.run` makes this setup opaque to the user until after execution, if ever.
        1. Information about the algorithm (target precision, iterations, number of
        ancillas, ...) and problem (Hamiltonian, cost function, ...) are not distinguished.
        
        As the `QuantumAlgorithm` primarily just enforces the presence of a `run(backend)` and provides the same base type for all algorithms -- which might not be necessary -- this raises the question: Do we really need it?

- What will the proposed changes enable?
  - Better hierarchical algorithm constructions - sub-alggorithms, etc. - and less hacking to implement these.
  - Better interchangeability and clearer result passing interfaces.
  - Using the same algorithm instance many times.
  - The main execution method(s) of an algorithm can be named for their specific (often industry-nomenclature) action.

- What will be the outcome?
  - Generally more flexibility in developing and using quantum algorithms.
  - Less re-defining of the same algorithm just for different problems.

- Who will benefit?
  - Any user of algorithms by having more precise names for the algorithm executions.
  - Quantum algorithm & application (QA&A) programmers, implementing more rigorous algorithms
    by having being more flexible in the development of quantum algorithms.
  - Newcomers: since quantum algorithms don't have this inherent difference to classical ones.
  - Code-checkers, since we don't need to re-define almost the same instance over and over again (and thus have fewer of code!)

## User Benefit
- Who are the target users of this work?
    - QA&A programmers
    - Aqua users in general

- How will users or contributors benefit from the work proposed?
  - Users
    -  More expressive and domain-friendly execution interfaces
        -  `solve(problem)` is a well-known and more natural execution structure in optimization
        - Train/Test/Predict or Fit/Test/Predict is a well-known and more natural execution structure in machine learning, and is used across Scikit-learn
  - Contributors:
    -  More flexibility in the definition of an algorithm
  - Both
    -  No need to re-define the same algorithm for different problems
    -  One less class to learn about, less wondering about where `run` actually comes from
    -  Easier to pass algorithm instances around and reuse existing algorithmic primitives
    
    
## Design Proposal

### Partial Construction and Component Reuse

With respect to reuse and partial construction, would ideally like to achieve the following:
```
algorithm = Algorithm(**some_settings)
for problem in problems:
    algorithm.run(problem)
```
This means moving the information about the backend into the initializer, and information about the problem from the initializer to `run` (or otherwise named execution function).

A consistent behavior should be enforced for component reuse within an algorithm. Specifically, we should enforce the following:
* Any component instance passed into the constructor or setter will be the default instance for that component. If an execution function is called, that instance will be used.
* For component instances passed to execution methods, e.g. `my_vpe.run(ansatz=my_ansatz)`, *where the execution function has been built to accept that component,* there are two options:
    * Option A: Any component instance passed into an execution function will override the default instance only for that execution. This allows alternate components to be tested during notebook or Python shell prototyping.
    * Option B: If an instance of a component was already set in the constructor and another is passed into the execution function, and error will be thrown.

**Suggestion:** We prefer option A, as 1) users calling execution functions with component instances generally understand that they are specifying that function to be run with that instance, so there is little ambiguity (how often will a user call `my_vpe.run(ansatz=my_ansatz)` and wonder why the ansatz passed in the constructor wasn't used?), and 2) option B will force more needless and error-prone calls like:
```
temp = my_vqe.ansatz
my_vqe.ansatz = new_ansat
my_vqe.get_min_eig()
my_vqe.ansatz = temp
```

#### Reuse Testing

Aqua's algorithms are not built for reuse today, and many store intermediate state which would be invalid upon re-initiation of the execution call. We must explicitly test reuse for *every algorithm* in Aqua, because even if only some algorithms have been built for reuse, users will have no way to know that others are not reusable, unless we introduce some awful `is_reusable` flagging. Where possible, all intermediate execution state should be confined to specific instance variables, ideally only `.result`, and wiped appropriately - see [Storing and Accessing Results](#storing-and-accessing-results) for detail about wiping state.

### Naming of `run`

Option A: Remove the abstract `run` method from `QuantumAlgorithm`, letting every algorithm decide how the main execution should be called. 

Option B: Keep `run` but let every algorithm define a more specific alias.

Option C: Keep `run` for some algorithms, but let other classes of algorithms inherit from base classes which define a more specific interface and no `.run` requirement (but keeping one in each algorithm for backward compatibility only).

**Decision in Aqua Review:** Option C. We may choose to add a deprecation message to `run` to tell the user what the new, changed behaviour is, allowing us to eventually remove it for the algorithms with improved interfaces.

### Do we need the `QuantumAlgorithm`?

| Pro                                  | Contra                                                           |
| :----------------------------------- | :--------------------------------------------------------------- |
| Uniform structure for all algorithms | Uniform structure is restrictive                                 |
| All algorithms sharing a base-type may come in handy later | Enforced structure is false for classical algorithms            |
|                                      | Not all algorithms should have to call the main method `run`     |
|                                      | Algorithms might have different requirements than `run(backend)` |

**Suggestion** Keep the base `QuantumAlgorithm`, but re-subclass certain algorithms as more meaningful base classes are implemented (see [Algorithm Base Classes and Alternate Execution Methods](#algorithm-base-classes-and-alternate-execution-methods) below). If all algorithms are accounted for and no use for the `QuantumAlgorithm` arises, we may deprecate it later.

### Algorithm Base Classes and Alternate Execution Methods

The following base classes could replace QuantumAlgorithm for some algorithms to provide more clarity and structure to the algorithm interfaces. Note that many algorithm will inherit from multiple base classes.
1. `VQAlgorithm` (already exists) - provides core structure and utilities for running variational optimization, so inheritors need only specify a cost function. Also supply valuable getters and setters, such as `.optimal_params`, `get_optimal_circuit`, `get_optimal_cost`, `.ansatz`, `.initial_point`, etc.
    1. Examples: VQE, VQC, QGAN, QAOA (via VQE)
1. `AmplitudeEstimationAlgorithm` (already exists) - provides functionalities to set and get the problem definition via the $A$, $Q$ and $S$ operators and ensure that all amplitude estimators have the same execution function, so they can easily be swapped for one another.
2. `MinEigenSolverAlgorithm` - Many algorithms require a MinEigenSolver as a subroutine, and an enforced interface would allow users to pass their eigensolver of choice into an application or meta-algorithm.
    1. Instance methods may include:
        1. `get_min_eigenvalue` - As described above, should return the result if the result has already
    2. Examples: VQE, QAOA, QPE, ExactEigensolver
3. `ClassificationAlgorithm` - TODO
    1. Examples: VQC, QSVM, Classical SVM
4. `LinearSolverAlgorithm`- TODO
    1. Examples: HHL, ExactLSsolver
5. `ExpectationAlgorithm`
    1. Examples: PauliExpectation, MatrixExpectation, AerExpectation
6. `EvolutionAlgorithm` - TODO
7. `VariationalOptimizer` - TODO
8. `GradientAlgorithm` - TODO
    1. Examples: ParameterShiftGradient, NumericalGradient

**Decision:** We should consider naming these to consistently end with "Algorithm" or an adjective, such as "Estimator" or "Solver."

### Algorithm Directory Hierarchy

 The present organization of the algorithm directory is unintuitive, and relies on naming conventions which are not industry standard in Quantum algorithms. We should update this to reflect a more intuitive structure. Some options:
1. Fully flat - all algorithms in one directory
1. Group by algorithm behavior - Variational, Classical, Oracular - and flat otherwise
1. Group by algorithm purpose - MinEigenSolver, LinearSolver, AmplitudeEstimation - and flat otherwise
1. Flat, except a few obvious exceptions - Classical, Oracular, AmplitudeEstimation - to keep things tidy

**Suggestion:** We suggest grouping algorithms by purpose (base class), to make obvious to users which algorithms can be interchanged, plus a directory for Oracular algorithms, as some of those are strictly educational. This allows users to clearly see that `ExactLSsolver` is available for comparison with `HHL`, which is very useful for new users who may not yet be aware of all of Aqua's algorithms. For algorithms which have multiple purposes, of which there are presently none, algorithms should be placed in the directory corresponding with the purpose for which they are more commonly used. 

### Storing and Accessing Results

It is often desirable for an algorithm to store additional result information after executing. For example, when running VQE, often the optimal parameters are also needed. In addition, often information about the computation itself is desirable, such as the number of iterations. In Python, generally results are not cached inside the execution object, but rather returned from execution functions as multiply-returned tuples and never stored. However, this structure would break the interchangeability of various algorithms in Aqua, as if VQE returned `(min_eigval, optimal_parameters)`, QPE or ExactEigensolver obviously would not do the same. This problem is solved in Aqua today by storing these values in a dictionary in a `.ret` instance variable, but does not enforce consistency between the algorithms and forces the user to inspect source code to understand how to access these values. So there are three core user journeys which must be met:
1. I am running VQE to study its behavior, and therefore want access to many other aspects of execution. I do not want to rerun the algorithm to access different computed properties. Sometimes I am forgetful while prototyping and realize that after I called `get_min_eigenvalue` that I also wanted the `optimal_params`.
1. I am an algorithm which requires an eigensolver as a subroutine (e.g. ADMM), and I want to simply call `get_min_eigenvalue`, without needing to branch for different multiply returned tuples or "return" objects from different algorithms. Sometimes I need to access different properties one after another, e.g. calling `get_eigen_circuit` after `get_minimum_eigenvalue`, and I don't want to have to rerun to do so.
1. I am an algorithm which requires a variational algorithm as a subroutine. I want to call `get_minimum_parameters` without needing to understand some `EigenSolverResults` object vs. a `VQCResult` object. Sometimes I need to access multiple properties, e.g. call `get_minimum_cost` and then `get_optimal_circuit`, and I don't want to have to rerun to do so.

We propose the following structure across Aqua and the App stacks:

* Heavy execution functions, such as `.solve` for the Optimization algorithms or `.find_minimum` for VQAlgorithms should return a result object, and store it in a `.result` instance variable.
* Getters for computed properties (e.g. `get_min_eigenvalue` or `.optimal_params`) should check whether `.result` contains the required property, and if not, execute. This is the structure already used by the `VQAlgorithm` base class. This way, the logic for how the data in the `.result` object are accessed can be contained in the algorithm.
    * This introduces a possible ambiguity where a user may change components of the algorithm and then call a getter for a computed property. Therefore, we should wipe the results object in the component setters so the computed property must be recomputed the next time it is requested. If the user wants to modify the components and then re-access a previously computed property for a set of component instances, this constitutes hacking and the user can access the private (underscore) component variables.

## Implementation procedure:

For each new interface base class (e.g. MinEigensolver), do the following: 
1. Implement the new base class, include the abstract execution interface methods, property setters and getters, and result getters.
1. Change the relevant algorithms to subclass from the new base instead of `QuantumAlgorithm`, and implement the required instance methods. Add `.run` as an abstract method.
1. Change the relevant algorithms to allow `None` specified arguments, which can be specified later in setters or execution functions.
1. Test algorithm reuse explicitly for each algorithm, especially checking state wiping after component changes.
3. Add a backend/QuantumInstance argument to into the constructors, initializing the QuantumInstance if needed, and testing that it behaves correctly.

Finally, test algorithm reuse for all remaining algorithms in Aqua.

Note, we should initially execute these changes without breaking backward compatibility (e.g. preserve shell `.run` methods, as below), and we can revisit a full `.run` deprecation at a later time. For example:
```
class MyAlgorithm:
    def __init__(self, settings,
                 problem=None,  # changed: problem specification is optional
                 backend=None, **backend_kwargs):  # added: backend info
        self._quantum_instance = ...
        self._algorithm_settings = ...
        self._problem_specifications = ...  # warning: set the problem using `solve`

    def solve(self, problem):
        return ...

    def run(self, backend, **kwargs):  # warning: use `solve` instead of run
        self._quantum_instance = ...
        return self.solve(self._problem_specifications)
```