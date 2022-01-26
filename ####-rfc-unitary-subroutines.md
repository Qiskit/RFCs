# Declaration syntax for more general unitary subroutines

| **Status**        | **Proposed**                                         |
|:------------------|:-----------------------------------------------------|
| **RFC #**         | ####                                                 |
| **Authors**       | Niel de Beaudrap (niel.debeaudrap@sussex.ac.uk),     |
| **Deprecates**    |                                                      |
| **Submitted**     | 2022-01-26                                           |
| **Updated**       | 2022-01-26                                           |


# Summary

We propose a new kind of subroutine for OpenQASM3, supplementing ``gate`` and ``def`` subroutines.
These are ``unitary`` subroutines, which correspond to 'circuit'-scale unitary operations on one or more quantum registers.
We define a syntax to define and invoke such ``unitary`` subroutines, describe how they differ from ``gate`` subroutines, and describe how they may be used in conjunction with the ``ctrl @`` and ``inv @`` gate modifiers.

# Motivation

It is useful to be able to define subroutines which act on aggregate quantum data, grouped into quantum registers.
This may be done in principle in OpenQASM3 using ``def`` subroutines, but doing so does not allow possibility of inverting these subroutines or controlling them coherently, even when the subroutine happens to consist entirely of unitary operations.
Such 'circuit'-level unitary subroutines are commonplace in the quantum literature.
Enabling such subroutines to be easily expressed will serve to increase the versatility of OpenQASM3 for programmers, and also for educators.
This is also a preliminary step towards making it possible to define _portable families_ of such operations, such as the quantum Fourier transform, through subroutine declarations.

The distinct features which we aim to capture with ``unitary`` subroutines, are:
 * in contrast to ``gate`` subroutines, they may take registers as arguments, and classical parameters other than angles;
 * also in contrast to ``gate`` subroutines, the subroutine definition may involve control flow using run-time constants;
 * in contrast to ``def`` subroutines, these subroutines may be inverted and coherently controlled using gate modifiers.
The intent is to provide a means of defining unitary subroutines of greater versatility than is possible with ``gate`` subroutines, while retaining some of the advantages of a subroutine in which there is a clear separation between classical program and quantum data (e.g., no measurement or feedback loops between the two kinds of data).
These subroutines would in principle be easily compiled to sequences of basic blocks similar to a top-level quantum program, even when invoked with modifiers such as ``ctrl @`` or ``inv @``.


# Proposed syntax, with examples

We present the proposed syntax below, with simple 'toy' examples which may be more typical of an educational setting rather than application.

## A simple unitary subroutine declaration

The proposed declaration syntax for ``unitary`` subroutines differs somewhat from that of gates, in that they may have quantum operands consisting of registers of any specified width (or arrays of qubits of some specified shape).
Because the types of the operands cannot be assumed to be individual qubits, they must be declared explicitly.
(In order to separate the subroutine name from the types of its quantum operands, we introduce a ``:`` as a delimiter between them.)
The unitary operation then acts on those operands, without use of broadcast semantics.
For example:

        // define a cyclic shift operation on
        // a three-dimensional subspace spanned
        // by |00>, |01>, |10> of a two-qubit state,
        // representing addition of one (mod 3) on
        // an encoded qutrit
        unitary qutritX : qubit[2] a {
          x a[1];
          cx a[1], a[0];
          cx a[0], a[1];
        }

        // define a ‘clock’ operator on a three-dimensional
        // subspace spanned by |00>, |01>, |10>, a two-qubit state
        unitary qutritZ : qubit[2] a {
          rz(2*pi/3) a[0];
          rz(-2*pi/3) a[1];
        }

In both cases the operand is a single register ``a`` consisting of two qubits.
(When invoked, the subroutines ``qutritX`` and ``qutritZ`` may act on a 'register' which is in fact a slice of some larger register.)

As the syntax may suggest, ``unitary`` subroutines may not involve any explicitly non-unitary operations such as ``reset`` or ``measure``.
Nor may they be defined using any subroutines apart from ``gate`` subrotuines, and other ``unitary`` subroutines.

## Unitary subroutine invocations, with and without gate modifiers

In addition to acting on quantum registers as operands, ``unitary`` subroutines may be controlled or inverted, using the same modifiers allowed for ``gate`` subroutines:

        // define a controlled-addition gate on two qutrits,
        // where each qutrit is encoded in two qubits
        unitary qutritCX : qubit[2] c, qubit[2] t {
          ctrl @ qutritX c[0], t;
          ctrl @ inv @ qutritX c[1], t;
        }
        
In this example, we define a ``unitary`` subroutine acting on two operands, each of which is a register of two qubits.
The unitaries are defined by modifications of the ``qutritX`` subroutine, in each case using the ``ctrl @`` modifier, which adds a single qubit to the beginning of the list of quantum operands, as with ``gate`` subroutines.
Using these declarations, we may then invoke ``qutritCX`` on the top level as follows:

        qubit[2] a;
        qubit[2] b;
        h a[0];
        qutritCX a, b;
        qutritZ a;

Note that the invocations of ``qutritCX`` and ``qutritZ`` are in effect similar in style to invocations of gates.
However, they do not involve any broadcast semantics.
In particular, invocations of these subroutines are only well-typed if they are invoked on quantum registers whose widths match the operands described in the respective subroutine declarations.

## Classical parameters and control flow in unitary subroutines

In common with ``gate`` subroutines, a ``unitary`` subroutine may also be defined using zero, one, or more classical parameters.
Also in common with ``gate`` subroutines, these parameters are specified in parentheses, before all of the quantum operands.
In contrast to ``gate`` subroutines, these parameters may have a type other than ``angle``, and so their types should be explicitly declared.
Also in contrast to ``gate`` subroutines, these classical arguments may be used for control flow statements in the body of the subroutine, also in contrast to ``gate`` subroutines.
For example:

        // define an operation which simulates adding an
        // integer (mod 3) to the state of a qutrit,
        // encoded in two qubits
        unitary qutritPlus (int b) : qubit[2] a {
          if (b % 3 == 1) qutritX a;
          else if (b % 3 == 2) inv @ qutritX a;
        }

The control flow allowed in the declaration of a ``unitary`` subroutine cannot involve any measurements, and therefore cannot involve any feedback between the classical parameters and the quantum operands.
In order to allow such a subroutine to be coherently controlled and inverted, we treat all parameters in the subroutine as being immutable.
It would also be allowed to define local variables within the scope of a ``unitary`` subroutine, as a convenience to the programmer, but these too would be immutable once initialised.
This behaviour extends also to the iterators of ``for`` loops, which may be used in the definition of a unitary subroutine:

        unitary QFT64 : qubit[6] q {
          uint n = 6;
          for j in [0 : n-1] {
            h q[j];
            for k in [j+1 : n-1]
              ctrl @ Rz(pi / 2**(k-j+1)) q[j], q[k];
            }
          for uint j in [0:n-1]
            if (j != n-1-j)
              swap q[j], q[n-1-j]
        }
        
While the iterators ``j`` and ``k`` above change their values in the course of their loops, we treat them as run-time constants.
This allows the inverse or a coherently controlled version of this same circuit to be easily computable from the subroutine definition.
Thus, using the above subroutine ``qutritPlus``, one may write a subroutine invocation as follows:

        input int s;
        qubit[2] t;
        qubit c;

        H c;

        ctrl @ qutritPlus(s) c, t;
        
Note that the additional quantum operand (a single qubit) introduced by the ``ctrl @`` modifier, occurs after the classical argument ``s`` to the subroutine ``qutritPlus``, as it would for a controlled ``gate`` invocation.
  
# Applications and future possibilities

The proposed feature of ``unitary`` subroutines stand on their own, in principle, as a feature which may be adopted for OpenQASM3.
This feature may be expected to be most useful to those who wish to write circuits with medium-to-large scale repeated structures, or who wish to write multiple programs involving a common medium-to-large scale subroutine.
A simple example would be defining non-trivial penalty functions in terms of boolean formulae on a register of more than a handful of qubits.

It is anticipated that ``unitary`` subroutines will be most useful in combination with a capacity to define 'generic' subroutines: this feature is to be proposed separately, and would be applicable both to ``unitary`` subroutines and ``def`` subroutines.
In particular, it would make it possible to define quantum Fourier transforms as a _family_ of unitary subroutines acting on quantum operands of various sizes, without the need to re-implement it for each operand size required in practise.
