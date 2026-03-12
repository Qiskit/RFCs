# Public Technical Vision for Qiskit

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | [Luciano Bello](https://github.com/1ucian0)  |
| **Submitted**     | 2025-09-09                                   |
| **Updated**       | 2026-03-11                                   |

## Summary

This RFC introduces a public _Technical Vision_ for Qiskit.
Such document defines the principles, values, and priorities that guide Qiskit's development, offering contributors and stakeholders a shared foundation for collaboration.
It is not a binding specification but a living statement of intent, designed to evolve as the project and ecosystem grow, with IBM Quantum's strategy also playing a guiding role.
By making this vision public, the goal is to foster alignment, transparency, and shared understanding across IBM Quantum, the Qiskit community, and the broader quantum computing software ecosystem.


## Motivation

There is a tradition of foundational documents in developer communities (for example, [Debian Free Software Manifesto](https://www.debian.org/doc/manuals/project-history/manifesto.en.html) and [OpenStack’s Technical Vision](https://governance.openstack.org/tc/reference/technical-vision.html)) that help participants align around shared values and a common direction.
In a similar spirit, this Qiskit Technical Vision defines the principles, commitments, and aspirations that currently guide Qiskit's development.

A shared technical vision provides clarity and alignment for everyone involved in Qiskit's development, including downstream projects.
It defines what Qiskit represents at a given point in time while outlining the direction it intends to take, helping to inform technical decisions.
By articulating guiding principles and priorities, the vision promotes transparency, strengthens collaboration, and supports coordination across IBM Quantum and the broader quantum computing community.
Maintaining this vision under version control also ensures that changes and their justifications are documented as part of a pull-request workflow, making its evolution explicit and traceable over time.

## User Benefit

The technical vision benefits a wide spectrum of Qiskit stakeholders: researchers, developers, educators, students, partners, and downstream projects.
Users gain a clearer understanding of Qiskit's scope and direction, which helps them plan, build, and sustain projects with confidence.
Contributors benefit from a shared framework that reduces ambiguity and aligns development choices across teams and organizations.
Together, these outcomes create a more predictable and trustworthy ecosystem where community members can invest their efforts effectively.


## Design Proposal



A possible initial version of the document is included [here](####-qiskit_technical_vision/TechnicalVision.md).
This instance serves a dual purpose: it exemplifies the level of detail expected of a *Qiskit Technical Vision* document, and it acts as the initial version of that document.

This RFC is fundamentally about **having** a public *Qiskit Technical Vision* document.
The specific content is secondary, but an initial version must be included to make the vision concrete from day one.

## Questions


**What is the scope of the *Qiskit Technical Vision* document? Which projects does it affect?**

The vision is for *Qiskit Projects* (usually living in the [Qiskit GitHub organization](https://github.com/Qiskit/qiskit/)) identifiable with the header `This code is a Qiskit Project`.

**What is the process to update the *Qiskit Technical Vision* document?**


The document is owned by Qiskit Project maintainers and lives in the [Qiskit organization profile](https://github.com/Qiskit/.github/).
To propose an update, a representative or maintainer of a Qiskit Project opens a pull request with the proposed change, explaining and justifying it.
After discussion in the PR (where questions from everyone are welcome, but only consensus among Qiskit Projects is required) the designated owner merges the change.

No RFC needed for updates to the document.

**How often is the *Qiskit Technical Vision* document updated?**

There is no fixed schedule.
Updates are made on demand when Qiskit Project maintainers determines that the vision has changed or is in the process of changing.

