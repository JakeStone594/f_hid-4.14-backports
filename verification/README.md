# Verification method

Three rules govern every result in this directory. They exist because each one caught a wrong
conclusion in this project before it shipped.

## 1. Single-variable regression

A fix is closed by rebuilding with exactly one thing changed — the patch — and re-running the same
repro under the same conditions. Conditions are stated explicitly and held: cable state, whether the
UDC is attached, charge state. If an intermediate state of the repro fails to reproduce identically
between builds, the run is void, because then more than one variable moved.

This is what makes a zero a *result* rather than an *absence*.

## 2. Positive controls

A count of zero is only evidence if the same run also produced a non-zero count for something that
must be there. Without that, "the bug is gone" and "the probe never ran" are the same log.

This is not hypothetical here. During this work a harness reported clean for every symbol it
checked, and had in fact never executed once — the state it was supposed to change had an identical
checksum before and after. Every measurement below therefore names its control.

## 3. Say what the evidence reached, and no further

The strongest temptation in this kind of work is to describe a mechanism one level deeper than the
measurement supports. It happened three times in this project. The discipline:

- **Measured:** what an instrument actually showed.
- **Inferred:** what follows, marked as inference.
- Where the two are conflated, a reader is invited to discount the measured part along with the
  speculative part — which is how a real, hardware-confirmed result gets thrown out with the story
  attached to it.

`03-cve-2026-31606.md` is the clearest application: the fix is correct by source reading, and this
device cannot confirm it. Both halves are stated.

## A trap specific to this kernel

**Kernel taint cannot be used as a health signal here.** The value is 516 before and after any of
these events, because bit 9 (`W`) is already set by a boot-time RCU warning. Any check keyed on
taint is therefore blind to every warning after the first of a boot. Counting matched fault
signatures in the log ring is the substitute used throughout.

Relatedly, `CONFIG_DEBUG_LIST=y` is the only reason defect 2 is visible at all — `SLUB_DEBUG` and
`DEBUG_OBJECTS` are both unset on this kernel. Drop `DEBUG_LIST` for size and that class of
corruption becomes silent rather than absent.
