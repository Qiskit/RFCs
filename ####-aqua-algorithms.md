# Features and necessity of the quantum algorithm type

| **Status**    | **Proposed**                              |
| :------------ | :---------------------------------------- |
| **RFC #**     | ####                                      |
| **Authors**   | First Author (first.author@ibm.com),  ... |
| **Submitted** | YYYY-MM-DD                                |
| **Updated**   | YYYY-MM-DD                                |

## Summary

Aqua's quantum algorithms are of type `QuantumAlgorithm`, which dictates that all algorithms
must are executed through a  `run` method whose arguments specify the backend used for circuit
execution.
This enforces several restrictions, such as not being able to use different backends or adaptive
settings. Further, the current structure makes passing algorithm instances to meta algorithms
difficult and hacky since once instanciated the problem the algorithm solves is usually fixed.

This RFC discusses the issues of the current `QuantumAlgorithm` and tries to assess, whether
this base type is still useful or if it might complicate the development of quantum algorithms.

## Motivation
- Why are we doing this?
    - Current structure:
        - To run a quantum algorithm, one initializes the algorithm with all necessary properties
        and then calls `run(backend)`, specifying a backend (or quantum instance) for the
        circuit execution. This method returns the algorithm result.
        - All quantum algorithms inherit from the abstract `QuantumAlgorithm` type, which requires
        an implementation of the actual algorithm execution via the `_run()` method.
        This method is called, when the user calls `run(backend)`.
        - Noteable facts:
          - All algorithms construct circuits
          - The problem the algorithm solves is defined in the initializer

        The execution is
        only allowed to take a `QuantumInstance` plus configuration.
        Thus, all configuration must take place before the execution, mostly in the initializer.
        In this execution, most (all?) algorithms construct circuits using the information provided
        in the initializer, execute all iterations, process the outcome and return the result.
    - Problems with that structure:
        1. Algorithms are fixed, the problems they solve are not. Why do we have to specify
        a new algorithm instance for each problem when we could just input the information when
        we actually run it.
        1. Missing flexibility: configuring the algorithm in the initializer makes the algorithm instance problem-specific. We are therefore unable to pass an algorithm instance that can solve different problems to meta algorithms. Examples: ADMM, any optimization, etc.
        1. The term "algorithm" is extremely broad. Classical algorithms don't share a common base
        class (compare Qiskit, scikit, etc.), why should quantum algorithms share one?
        1. Naming: every single quantum algorithm has is executed using `run`, though other names
        can be much more expressive. Optimizers `optimize` or `minimize` a quantity, neural networks `learn`, HHL `solve`s, etc. Classical algorithms do not have this naming restriction,
        why should quantum algorithms have them?
        1. If an algorithm runs and compares multiple backends or settings, this is not supported in
        a convenient manner.
        1. Algorithms can be hardware specific, the backend must be known beforehand and it doesn't
        make sense to provide the backend in `run`.
        1. Information about the algorithm information (target precision, iterations, number of
        ancillas, ...) and problem information (Hamiltonian, cost function, ...) are not distinguished.
    - These problems are summarized in the following concerns:
        1. The `run` method is restrictive for the huge variety of quantum algorithms.
        2. An algorithm instance should solve different problems.
        3. The backend should not be required in `run`.

        As all the `QuantumAlgorithm` does is requiring a `run(backend)` and providing the same base type for all algorithms -- which might not be necessary -- this raises the question:
        Do we really need it?

- What will this (= lifting the restrictions of `QuantumAlgorithm`) enable?
  - Use the same algorithm instance to solve multiple problems.
  - The main method of an algorithm can be named after its actual action.
  - Multiple backends may be used within an algorithm.

- What will be the outcome?
  - Generally more flexibility in developing and using quantum algorithms.
  - Less re-defining of the same algorithm just for different problems.

- Who will benefit?
  - Any user of algorithms by having more precise names for the algorithm executions.
  - Quantum algorithm & application (QA&A) programmers, implementing more rigorous algorithms
    by having being more flexible in the development of quantum algorithms.
  - Newcomers: since quantum algorithms don't have this inherent difference to classical ones.
  - Code-checkers, since we don't have to re-define almost the same instance over and over again (and thus have less lines of code!)

## User Benefit
- Who are the target users of this work?
    - QA&A programmers
    - Aqua users in general

- How will users or contributors benefit from the work proposed?
  - Users
    -  `solve(problem)` is a well-known instruction
    -  more expressive names
    -  no need to re-define the same algorithm for different problems
    -  one less class to learn about, less wondering about where `run` actually comes from
    -  easier to pass algorithm instances around
  - Contributors:
    -  no need to re-define the same algorithm for different problems
    -  one less class to learn about, less wondering about where `run` actually comes from
    -  easier to pass algorithm instances around
    -  more flexibility in the definition of the algorithm

## Design Proposal

### One-instance-many-problems-design

We would like to do
```
algorithm = Algorithm(**settings)
for problem in problems:
    algorithm.run(problem)
```
This means moving the information about the backend into the initializer,
and information about the problem from the initializer to `run`.
Can this be done? See open questions.

### Naming of `run`

**A.** Remove the `run` method from `QuantumAlgorithm`, let every algorithm decide how the main
execution should be called.

**B.** Keep `run` but let every algorithm define a more specific alias.

**Suggestion** The added confusion by adding an alias seems not worth keeping `run`.
In any case: Keep a deprecated `run` and tell the user what the new, changed behaviour is.

### Do we need the `QuantumAlgorithm`?

| Pro                                  | Contra                                                           |
| :----------------------------------- | :--------------------------------------------------------------- |
| uniform structure for all algorithms | uniform structure is restrictive                                 |
| all algorithms share same base-type  | enforced structure: not true for classical algorithms            |
|                                      | not all algorithms should have to call the main method `run`     |
|                                      | algorithms might have different requirements than `run(backend)` |

**Suggestion** Remove the base type `QuantumAlgorithm`. If several algorithms share the same base,
create a base type for this, such as already existing for `VQAlgorithm` or `AmplitudeEstimationAlgorithm`.
This is not a change users note.
To maintain backward compatibility and to warn the users, keep `run` in the algorithms but deprecate it.

## Detailed Design

Implementation procedure:
1. deprecate the class initializer of `QuantumAlgorithm`
2. extend `_run` to `run` in all algorithms, if a better name exists, rename `run` and deprecate it
3. move the backend information into the initializer, and the problem information to the execution function

Note, that these changes are possibly without breaking backward compatibility.
For example:
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


## Questions
- Why was the information about the backend in `run`? GUI? Can it be moved to the initializer?