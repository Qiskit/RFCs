# Qiskit Pulse: Relative timing for Schedules

| **Status**        | **Proposed** |
|:------------------|:---------------------------------------------|
| **RFC #**         | TBD                                          |
| **Authors**       | Lauren Capelluto                             |
| **Submitted**     | 3/11/20                                      |
| **Updated**       | ---                                          |


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
 - Pulse users (primarily, experimental physicists interested in using IBM backends) should get a performance improvement.
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

### Notation
Here we will use the following notation, where `self` is the `Schedule` initiating the operation in question:

 - |C| : the total number of channels in `self`
 - N<sub>self</sub> : the total number of instructions in `self`
 - N<sub>new</sub> : the total number of instructions in another `Schedule` being added in some way to `self`
 - N<sub>C</sub> : the number of instructions on channel C within `self`

### New Schedule methods

| Method        | Runtime        |  Description     |
|---------------|----------------|------------------|
|`barrier(self, channels=None)`| O(\|C\|) | Add `Delay`s across `channels` so that all of the `channels` are occupied until `self.duration(channels)`. By default, barrier on all channels. |
|`left_align(self)` | O(\|C\|) | This is the default behavior. Return `self`.
|`center_align(self)` | O(\|C\|) | Pre- and append equally sized `Delay`s (how to split odd durations?) on each channel so that all channels are occupied until `self.duration()` |
|`right_align(self)` | O(\|C\|) | Prepend `Delay`s on each channel so that all channels are occupied until `self.duration()`.

**Example**

```
sched = Schedule()
sched += Delay(10, drive0)
sched += Play(gaussian, drive0)
sched = sched.barrier(drive0, drive1)
subsched = Play([1.0]*20, drive1)
subsched = Play([1.0]*10, drive0)
sched += subsched.right_align()
```

### API Changes

| Method        | Runtime        |  Change          |
|---------------|----------------|------------------|
| `append(self, schedule)` | O(\|C\|) | Previously, `append` would add `schedule` to `self` with relative times preserved, at time set by `self.duration({schedule.channels}.intersection({self.channels}))`. The new method will simply extend the instructions in `self` with those in `schedule` on a per channel basis. |
|`insert(self, time, schedule)`| O(N<sub>new</sub> * N<sub>C</sub>) | This will be slow generally, but it can be implemented with the same runtime as `append` when the `time` is greater than `self.duration(schedule.channels)`. |

**Example**

```
# To recreate previous append behavior:
sched = sched.barrier(sched2.channels)
sched += sched2
```

## Detailed Design

The `Schedule` will have an internal attribute, `_insts_by_chan`:

```
_insts_by_chan: Dict[Channel, List[ScheduleNode]]
```

**Example**
```
# sched._insts_by_chan
{
    DriveChannel(0):
        ('instructions'=[Delay(duration=10, ...),
                         Play(duration=100, ...),
                         Play(duration=20, ...)],
         'duration'=130),
     DriveChannel(1):
        ('instructions'=[Delay(duration=110, ...),
                         Play(duration=20, ...)],
         'duration'=130),
    ...
}
```

A rough sketch for the implementation of some of the methods described above could look like this:

```
# These can be done with a few more lines to avoid mutating the input, but for simplicity, I'll leave it like this to demonstrate the main idea

    def append(self, schedule):
        for chan, data in schedule._insts_by_chan:
            self._insts_by_chan[chan].instructions.extend(data.instructions)
            self._insts_by_chan[chan].duration += data.duration

    def barrier(self, channels):
        total_duration = self.ch_duration(channels)
        for chan in channels:
            delay_time = total_duration - self._insts_by_chan[chan].duration
            self._insts_by_chan[chan].instructions.append(Delay(duration=delay_time))
            self._insts_by_chan.duration += delay_time

    def right_align(self):
        for chan, data in self._insts_by_chan:
            data.instructions = [Delay(self.duration - data.duration)] + data.instructions
            data.duration = self.duration
```

 
## Alternative Approaches


## Questions

 1. Should we create `insert(self, index, channel)`?
 1. We can support the current `insert` method, but the runtime with be O(|N<sup>2</sup>|). Are we committed to discouraging its use?
 2. Something like `sched |= sched << time` is *much* less efficient than `sched.insert(time, sched)` because the shift happens separately from the union. I don't know how to fix this.

## Future Extensions

A related and important companion to this RFC is the implementation of a schedule building interface. See a sketch of how this might look from our in-house hackathon project lead by @taalexander: [branch here](https://github.com/taalexander/qiskit-terra/tree/pulse-builder-interface) and [demo here](https://github.com/taalexander/qiskit-terra/blob/pulse-builder-interface/notebooks/Pulse%20DSL%20Demo.ipynb).

Altogether, this relative timing feature, the above builder, and the [instruction/command unification work](https://github.com/Qiskit/rfcs/pull/12), we will have migrated Pulse towards a very user friendly and physicist-charming interface, paving the way for wonderful Pulse-based work to be implemented in Ignis.

Additionally, as stated in the motivation, this work will be crucial for supporting general control flow in Qiskit Pulse.
