# [WIP] Grouping Pauli strings with entangled measurements

| **Status**     | **Draft**                              |
|:-------------- |:-------------------------------------- |
| **RFC #**      | ####                                   |
| **Authors**    | Takashi Imamichi (imamichi@jp.ibm.com) |
| **Deprecates** | RFC that this RFC deprecates           |
| **Submitted**  | 2020-02-07                             |
| **Updated**    | 2020-02-07                             |

## Summary

This RFC proposes grouping of Pauli strings with entangled measurements [1].

- [1] Ikko Hamamura, Takashi Imamichi, "Efficient evaluation of quantum observables using entangled measurements," [arXiv:1909.09119](https://arxiv.org/abs/1909.09119).

## Motivation

- Qiskit Aqua provides a function of grouping Pauli strings with tensor product basis (TPB), i.e., [TPBGroupedWeightedPauliOperator](https://github.com/Qiskit/qiskit-aqua/blob/9f5397761f96af0545938c04e7ed1cc65a9886cf/qiskit/aqua/operators/tpb_grouped_weighted_pauli_operator.py#L39)
  - It decrease the number of iterations to evaluate the expectation value of a qubit Hamiltonian
- The paper [1] showed that grouping with TPB and entangled measurements can generate smaller number of groups.

## User Benefit

- People who evaluate expectation value of qubit Hamiltonians with real devices or qasm_simulator.
- This RFC reduces the number of iterations compared with TPB grouping.

## Design Proposal

Pauli string is a tensor products of Pauli operators, e.g., IZXI = I⊗Z⊗X⊗I.
Pauli strings XI and IZ can be grouped with a measurement (1: X, 2: Z),
i.e., first and second qubits are measured by X and Z measurements, respectively.

Grouping of Pauli strings consists of following input and output.

Input:
- A set of measurements to be used for the grouping
  - For grouping with TPB, there are three types of measurements ({I, X}, {I, Y}, {I, Z}) available.
  - For grouping with Bell measurements, there in one type of measurement ({II, XX, YY, ZZ}) available.
  - For grouping with TPB and Bell measurement, there are four types measurements ({I, X}, {I, Y}, {I, Z}, {II, XX, YY, ZZ}) available.
- Grouping algorithm
  - Aqua provides largest-degree-first-coloring (LDFC) algorithm for grouping with TPB.
  - A greedy grouping algorithm for grouping with entangled measurements is proposed in [1].

Output:
- Groups of Pauli strings and measurements associated with each group
  - A measurement associated with a group of Pauli strings assign given measurements without overlap.
  e.g, Pauli strings XYXI and ZIZX can be grouped with measurement (1&3: Bell, 2: Y, 4: X),
  i.e, first and third qubits are measured by Bell measurements.
  Second and fourth qubits are measured by Y and X measurements, respectively.
- Quantum circuits corresponding to the measurements of the groups.


I propose `GroupedWeightedPauliOperator` that has the same interface as `TPBGroupedWeightedPauliOperator`.
`GroupedWeightedPauliOperator` receives measurement types and a grouping algorithm as parameters of the constructor
and generates groups.


## Detailed Design

TBD

Technical reference level design. Elaborate on details such as:

- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.

## Questions

Open questions for discussion and an opening for feedback.

## Future Extensions

- Tune the parameters of grouping based on backend information
  - For exmaple, if two qubit gate error is large, it is good to avoid using entangled measurements.

