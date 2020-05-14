# BackendConfiguration.supported_instruction

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Luciano Bello (luciano.bello@ibm.com)        |
| **Deprecates**    | nothing                                      |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |


## Summary
`BackendConfiguration` object has the attribute `supported_instruction` to report supported instructions to the transpiler or other elements, similarly to `basis_gates`.

## Motivation
- Why are we doing this? Right now, [some transpilation passes artificially handle instructions](https://github.com/Qiskit/qiskit-terra/blob/948671efd960d44597f8fc40094985cccf97e5d8/qiskit/transpiler/passes/basis/unroller.py#L57). That goes against the "backend agnostic" principle in Qiskit. 
- What will this enable? Backends will be enabled to extend or reduce the supported instructions. The transpiler could skip certain passes if we know that, for example, a state vector simulator does not have `measure`.
- What will be the outcome? A new attribute for the `BackendConfiguration`, called `supported_instruction`
- Who will benefit? Provider developers, backend maintainers, and transpiler pass developers.

## User Benefit
- Who are the target users of this work? Provider developers, backend maintainers and transpiler pass developers.
- How will users or contributors benefit from the work proposed?
Qiskit will be more flexible to support multiple backends.

## Design Proposal
The backend configuration should report the supported instructions (such as `measure`, `reset`, `barrier`, `snapshot`, and `if`). A specification on the possible values and their meaning is pending.


## Detailed Design
- Implementation procedure
	1. Add in Qiskit at backend configuration loading time if the `supported_instruction` is present. If not, raise a warning that the field will be required soon.
	2. Extend providers to include this attribute.
	3. Maybe extend the configuration backend on server side with this attribute (see Alternate Approaches)
	4. Once the warning period is due, remove the warning for a fatal error if the backend configuration attribute is not present.
	5. Move the hardcoded in transpiler passes (and other parts of qiskit-terra) by getting the list from this attribute. 
- Interaction with other features
	- providers need to be updated
	- terra needs to be updated
	- maybe backend configuration on server side (see Alternate Approaches).

## Alternative Approaches
There is no need to extend backend configuration reported by the server. The provider "knows" the real implementation of the backend and it has the authority to decide what should be included in this list.

A possible alternative is to extend `basis_gates`. That requires to bend the meaning of basis gate a bit too much.

## Questions
Open questions for discussion and an opening for feedback.

## Future Extensions
A list of all the instructions supported by Qiskit (with their meaning) needs to be compiled. (BTW, same applies to `basis_gates`)