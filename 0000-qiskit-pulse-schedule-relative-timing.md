# Qiskit Pulse: Relative timing for Schedules

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | TBD                                          |
| **Authors**       | Lauren Capelluto                             |
| **Submitted**     | see github                                   |
| **Updated**       | see github                                   |


## Summary
Update the internal structure of `Schedule`s to track instructions sequentially. `Instruction`s will not be played at an explicit time; rather, they will begin immediately after the last is done. To schedule idle time between instructions, `Delay` instructions must be used.

The user API will be updated to take advantage of this relative timing, with several new methods: `barrier`, `left_align`, `center_align`, and `right_align` (see: [new methods](#new-schedule-methods)).

## Motivation
(This section was copied from [this issue](https://github.com/Qiskit/qiskit-terra/issues/3749), written by @taalexander.)
Currently, all `pulse.Instructions` are stored in a `Schedule` with a two-tuple containing the `(start_time, Instruction)`. While this makes it clear the explicit starting time of every pulse it leads to convoluted internal data structures such as `TimeslotCollection` for tracking the time of all instructions.

It also encourages inefficient operations such as taking the union of two schedules. At first glance this operation should be efficient since all one needs to do is specifying the starting time of the instructions, but this requires verification that no instructions overlap in time in the two schedules resulting in expensive checks. The requirement for such a check to merge two schedules is obvious within the relatively schedules model, as one must look to make sure a delay exists in the first schedule for all instructions of the second schedule to merge.

Furthermore, explicit timing of instructions will become confusing with the introduction of control-flow to quantum programs as it would become difficult to program experiments containing loops that are not unrollable.

We should move to a model where instructions are explicitly scheduled with variable-duration `Delay`s (think NOPs) of a given duration. This will:

 - Make appending two schedules a much simpler operation as the dependency of instructions will be specified by their order on operands
 - This will allow a simple transformation to a DAG representation of a basic pulse schedule block
 - Have positive performance impacts.
 - Have implications for the current pulse API.

## User Benefit
 - Pulse users (primarily, experimental physicists interested in using IBM backends) should get a perforance improvement.
 - When we begin implementing control flow around Schedules, we will find this new implementation much easier to work with. This benefits frontend and backend developers, which in turn will give our end users new features sooner.

## Design Proposal

<!-- This is the focus of the document. Explain the proposal from the perspective of
educating another user on the proposed features.

This generally means:
- Introducing new concepts and nomenclature
- Using examples to introduce new features
- Implementation and Migration path with associated concerns
- Communication of features and changes to users

Focus on giving an overview of impact of the proposed changes to the target
audience.

Factors to consider:
- Performance
- Dependencies
- Maintenance
- Compatibility -->

### Notation:
I am going to use this notation here:

 - |C| : the total number of channels in `self` (of type `Schedule`)

 |N<sub>self</sub>|
 |N<sub>other</sub>|

### New Schedule methods

 - `barrier(self, channels)`: O(|C|)

   Add `Delay`s across `channels` so that all of the `channels` are occupied until `self.duration(channels)`.

Should the following take `channels`, or should we "keep it simple, stupid"? A user could always build a subschedule of the channels, or use filter to get the schedule they want....

| Method        | Runtime        |  Description     |
|---------------|----------------|------------------|
|`left_align(self)` | O(\|C\|) | This is the default behavior. We could have it return `self`, or we could remove any pre-pended `Delay`s. The latter is probably better, because `schedule.right_align().left_align()` should yield the original `schedule`. Should we also append `Delay`s? Likely. *Needs more thought* |
|`center_align(self)` | O(\|C\|) | Pre- and ap-pend equally sized `Delay`s (how to split odd durations?) on each channel so that all channels are occupied until `self.duration()` |
|`right_align(self)` | O(\|C\|) | Prepend `Delay`s on each channel so that all channels are occupied until `self.duration()`. Should we also remove trailing `Delay`s? |


### API Changes

 - `append(self, schedule)` O(|N<sub>new</sub>|)
 - `insert(self, time, schedule)`
 - `union` -> I still think this should be deprecated (redundant and usage is especially not encouraged for this)

## Detailed Design
<!-- Technical reference level design. Elaborate on details such as:
- Implementation procedure
  - If spans multiple projects cover these parts individually
- Interaction with other features
- Dissecting corner cases
- Reference definition, eg., formal definitions.
 -->
How internal data will now be tracked

How new and existing methods will be updated

 
## Alternative Approaches
<!-- Discuss other approaches to solving this problem and why these were not
selected.
 -->

## Questions
<!-- Open questions for discussion and an opening for feedback. -->
 1. `insert`
 1. Removing leading/trailing `Delay`s on `left_align`/`right_align`, respectively.
 1. Issue with not breaking `x += y << t`:

    `shift` will be easy to implement, but relative timing makes this impossible: `sched += sched2 << time`!! Should we kill shift?

 1. Should we create `insert(self, index, channel)`?

## Future Extensions

A related and important companion to this RFC is the implementation of a schedule building interface. See a sketch of how this might look from our in-house hackathon project lead by @taalexander: [branch here](https://github.com/taalexander/qiskit-terra/tree/pulse-builder-interface) and [demo here](https://github.com/taalexander/qiskit-terra/blob/pulse-builder-interface/notebooks/Pulse%20DSL%20Demo.ipynb).

Altogether, this relative timing feature, the above builder, and the [instruction/command unification work](https://github.com/Qiskit/rfcs/pull/12), we will have migrated Pulse towards a very user friendly and physicist-charming interface, paving the way for wonderful Pulse-based work to be implemented in Ignis.

Additionally, as stated in the motivation, this work will be crucial for supporting general control flow in Qiskit Pulse.
