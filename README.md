# Logical Equivalence Checks with CIRCT
[Google Summer of Code](https://summerofcode.withgoogle.com/) is a program for promoting open-source development to new contributors. I applied to this year's edition after noticing Fabian Schuiki's project idea for the development of a _logical equivalence checker for hardware designs_ within the Free and Open Source Silicon Foundation's [website](https://www.fossi-foundation.org/fossi).

Since I am interested in formal methods, I thought of it as a good opportunity to explore some aspects of formal verification in the context of a real-world application and, as I had no prior experience with hardware design, it would
act as a learning primer for that subject as well.

After a brief exchange with Fabian Schuiki and Jonathan Balkind from FOSSi, I redacted and submitted my project proposal, which then got accepted. This is the final report for the work I've done this summer.

## Motivation and project description
The [CIRCT](https://circt.llvm.org/) project is an open-source effort to develop Circuit Intermediate Representations, Compilers, and Tools by applying the [MLIR](https://mlir.llvm.org/) and [LLVM](https://llvm.org/) development methodology and best practices to the domain of hardware design tools.

In particular, it could act as a common platform and accelerate the development of interoperable design and verification tools, as opposed to today’s landscape of proprietary tools implementing their own disjoint and incompatible IRs.

The project idea was to demonstrate said verification prowess by implementing `circt-lec`, a basic LEC (Logical Equivalence Checker) for combinational CIRCT designs. Specifically, it has to translate two circuits into their fundamental boolean equations and formally prove or disprove their equivalence through the aid of an existing SMT solver (an engine for determining whether mathematical formulas are satisfiable).

## Final work
The project succeeded in meeting its goal of building a combinational circuit equivalence checking tool for CIRCT, covering all operations of the `comb` dialect and a few of the `hw` dialect.

Development was done on a forked repository ([url](https://github.com/frog-in-the-well/circt)), from which a final pull request was submitted ([url](https://github.com/llvm/circt/pull/3991)).

In addition, some presentations were held throughout the course of the project and detailed its advancements:
- at ~50% completion, I introduced the project during a [CIRCT weekly meeting](https://docs.google.com/document/d/1fOSRdyZR2w75D87yU2Ma9h2-_lEPL4NxvhJGJd-s5pk) to collect feedback from the community and spur discussion about formal methods;
- at ~75% completion, I presented a poster ([download](./assets/circt-lec%20poster.pdf)) at the [VTSA 2022](https://resources.mpi-inf.mpg.de/departments/rg1/conferences/vtsa22/) summer school, where I engaged with the verification research community and promoted the CIRCT project.

## Development rationale
Originally, the project aimed at developing a LEC for [LLHD](https://llhd.io/), a novel multi-level Intermediate Representation (IR) for Hardware Description Languages. 

But since LLHD was in the process of being merged as a CIRCT dialect, we concluded it would be better to instead write `circt-lec` for the basic CIRCT dialects, which every other CIRCT IR can transform into. This would contribute to the whole ecosystem because every design would become checkable.

#### SMT solver
I decided to employ [Z3](https://github.com/Z3Prover/z3) for the logical backend because it is a popular state-of-the-art SMT solver with many features, like proof production and tactic definition, which means more flexibility and better documentation. It also provides C++ bindings, which is the language in use in CIRCT and what `circt-lec` had be written in. Besides, being already a dependency of LLVM, it was the most natural choice.

#### Data-type support
To speed up development, we decided internally to represent all values as bitvectors. This can be easily done for integer values and simplifies performing operations Z3 as there would be no need to deal with type conversions, which could cause nasty runtime errors.

Ulterior data-types like arrays and structs were ignored because, if required one could always write a pass to transform them into their integer components, so they were not fundamental to have.

#### Regression tests
Because this tool is to be used to prove correctness, it is vastly important for it to be correct itself. While tests are no formal proof, because the tool asserts the equivalence of circuits for all possible values of inputs and outputs, hence all corner cases, I would argue testing a high-level operation against its gate-level decomposition projects a high degree of confidence the operation is implemented correctly.

For this reason, I wrote tests for many of the implemented operations using LLVM's own testing suite. As a further benefit, it has become easy to detect when breaking changes are introduced.

It has to be noted that SMT solvers have many bugs themselves which might cause reporting incorrect results.

#### Multi-value logic
While presenting to the CIRCT developers community, the issue of `comb`'s representation of multi-valued logic arose.

After that, a new attribute was added to each of the dialect's operations but, because there was still no consensus on how to satisfiably model it, we concurred to not engage with it in `circt-lec` while it is still unstable.

## An example of equivalence checking
Say we want to implement a [subtractor](https://en.wikipedia.org/wiki/Subtractor) for 8-bit integers.

We can start by defining a _half-subtractor_ which outputs the difference and borrow of two input bits.
```MLIR
hw.module @halfSubtractor(%in1: i1, %in2: i1) -> (borrow: i1, diff: i1) {
  %diff = comb.xor %in1, %in2 : i1
  %not_in1 = hw.instance "n_in1" @not(in: %in1: i1) -> (out: i1)
  %borrow = comb.and %not_in1, %in2 : i1
  hw.output %borrow, %diff: i1, i1
}
```

A _full-subtractor_ can then be written as a combination of two half-substractors; it returns the difference and borrow of subtracting two input bits while accounting for a possible previous borrow.
```MLIR
hw.module @fullSubtractor(%in1: i1, %in2: i1, %b_in: i1) -> (borrow: i1, diff: i1) {
  %b1, %d1 = hw.instance "s1" @halfSubtractor(in1: %in1: i1, in2: %in2: i1) -> (borrow: i1, diff: i1)
  %b2, %d_out = hw.instance "s2" @halfSubtractor(in1: %d1: i1, in2: %b_in: i1) -> (borrow: i1, diff: i1)
  %b_out = comb.or %b1, %b2 : i1
  hw.output %b_out, %d_out: i1, i1
}
```

Subtraction between two 8-bit input integers can then be implemented by composiing full-subtractors for each corresponding couple of bits from the inputs, while carrying the borrow from the previous subtraction. The resulting difference is the concatenation of every difference bit.
```MLIR
hw.module @completeSubtractor(%in1: i8, %in2 : i8) -> (out: i8) {
  %in1_0 = comb.extract %in1 from 0 : (i8) -> i1
  %in1_1 = comb.extract %in1 from 1 : (i8) -> i1
  %in1_2 = comb.extract %in1 from 2 : (i8) -> i1
  %in1_3 = comb.extract %in1 from 3 : (i8) -> i1
  %in1_4 = comb.extract %in1 from 4 : (i8) -> i1
  %in1_5 = comb.extract %in1 from 5 : (i8) -> i1
  %in1_6 = comb.extract %in1 from 6 : (i8) -> i1
  %in1_7 = comb.extract %in1 from 7 : (i8) -> i1
  %in2_0 = comb.extract %in2 from 0 : (i8) -> i1
  %in2_1 = comb.extract %in2 from 1 : (i8) -> i1
  %in2_2 = comb.extract %in2 from 2 : (i8) -> i1
  %in2_3 = comb.extract %in2 from 3 : (i8) -> i1
  %in2_4 = comb.extract %in2 from 4 : (i8) -> i1
  %in2_5 = comb.extract %in2 from 5 : (i8) -> i1
  %in2_6 = comb.extract %in2 from 6 : (i8) -> i1
  %in2_7 = comb.extract %in2 from 7 : (i8) -> i1
  %b0, %d0 = hw.instance "s0" @halfSubtractor(in1: %in1_0: i1, in2: %in2_0: i1) -> (borrow: i1, diff: i1)
  %b1, %d1 = hw.instance "s1" @fullSubtractor(in1: %in1_1: i1, in2: %in2_1: i1, b_in: %b0: i1) -> (borrow: i1, diff: i1)
  %b2, %d2 = hw.instance "s2" @fullSubtractor(in1: %in1_2: i1, in2: %in2_2: i1, b_in: %b1: i1) -> (borrow: i1, diff: i1)
  %b3, %d3 = hw.instance "s3" @fullSubtractor(in1: %in1_3: i1, in2: %in2_3: i1, b_in: %b2: i1) -> (borrow: i1, diff: i1)
  %b4, %d4 = hw.instance "s4" @fullSubtractor(in1: %in1_4: i1, in2: %in2_4: i1, b_in: %b3: i1) -> (borrow: i1, diff: i1)
  %b5, %d5 = hw.instance "s5" @fullSubtractor(in1: %in1_5: i1, in2: %in2_5: i1, b_in: %b4: i1) -> (borrow: i1, diff: i1)
  %b6, %d6 = hw.instance "s6" @fullSubtractor(in1: %in1_6: i1, in2: %in2_6: i1, b_in: %b5: i1) -> (borrow: i1, diff: i1)
  %b7, %d7 = hw.instance "s7" @fullSubtractor(in1: %in1_7: i1, in2: %in2_7: i1, b_in: %b6: i1) -> (borrow: i1, diff: i1)
  %diff = comb.concat %d7, %d6, %d5, %d4, %d3, %d2, %d1, %d0 : i1, i1, i1, i1, i1, i1, i1, i1
  hw.output %diff : i8
}
```

As it can be seen, implementing even simple operations in hardware can readily become complicated and induce implementation bugs.

In the case of subtraction, a first-order operation is already available so we could have just used `comb.sub` instead, like so:
```MLIR
hw.module @subtractor(%in1: i8, %in2: i8) -> (out: i8) {
  %diff = comb.sub %in1, %in2 : i8
  hw.output %diff : i8
}
```

But we might still be interested in checking whether our previous implementation is equivalent to the provided one. To do so we could invoke:
```bash
$ circt-lec input.mlir -c1=subtractor -c2=completeSubtractor
```
where `input.mlir` is a file collecting all the previous circuit definitions, and indeed the tool would report them as logically equivalent.
```
c1 == c2
```

## Future challenges
In the short term, I will follow the PR's discussion and lead it to a successful merge in the codebase.

While the tool as delivered is functional and usable, its scope is too big and complex to render it state-of-the-art within the twelve weeks of a GSoC project.

The following are some ideas for how to continue its development and improve it
which we collected but were unable to act on because of time constraints:
- finish implementing regression tests for those operations that miss them.
- Completing support for the `hw` dialect, which would allow any combinational CIRCT design to be checkable by `circt-lec` by first lowering it through `circt-opt`.
- Adding support for the `seq` dialect, thus extending the checking also for sequential circuits which involve registers. This could be achieved in two manners:
    - via Finite State Machine equivalence with bounded model checking, as otherwise, it would incur in memory problems; it isx complex to implement and would require asking the user to opt-in for sound but incomplete solutions, but it would also consent to check arbitrary properties in temporal logic.
    - Via Register correspondence, that is discovering the register pairs which are equivalent either through heuristics or simulation, thus reducing the problem to the equivalence of combinational circuits which the tool can already solve.
- In case of multiple outputs of which only a portion is wrong, showing information on just those values which are affecting the relevant outputs rather than the whole model.
- Showing only the interpretation of values at module boundaries rather than the whole model.
- Localizing the introduced bugs by computing the Craig interpolant between the circuits.
- Allowing users to submit a list of instances identifiers to be considered equivalent (e.g. in case of big designs with localized changes), vastly reducing the time to solve the equivalence check.
- Extending the tool to support different logical engine backends, either by integrating an abstract SMT API like Smt-Switch or by employing the SMT-LIB format internally.
- Adding support for multi-valued logic as used in SystemVerilog and VHDL.

Those willing to contribute should feel free to reach out to me, the email can be found in the poster.

## Acknowledgements
First of all, I need to thank my mentors Fabian Schuiki and Martin Erhart: their advice and expertise helped steer the project in the right direction and keep it on track.

Thank you to Jonathan Balkind for accommodating my other commitments during the summer by extending the deadlines and for overseeing this project in particular besides chairing all other GSoC activities within FOSSi.

A final shout-out to CIRCT's community for being receptive and welcoming; also to Google for organizing the GSoC program.
