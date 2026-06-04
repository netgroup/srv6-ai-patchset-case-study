# srv6-ai-patchset-case-study

Public analysis of an AI-generated patchset submitted to the Linux kernel SRv6 subsystem: what went wrong, why, and lessons learned for using AI in kernel development.

## Disclaimer

We are not here to blame anyone. We are here to learn and to improve the kernel development process, taking into account both the opportunities and the risks of AI.

There are currently no publicly agreed rules on the use of AI in kernel development — not even established best practices. So no one can be blamed for getting it wrong.

On the other hand, there is something that is clearly unsustainable: producing a 5000 lines-of-code patchset with AI may take 1 or 2 days of work, while manually reviewing it takes a Linux kernel maintainer 5 to 10 times longer. This asymmetry between the cost of generating code and the cost of reviewing it is at the core of the problem we want to address.

An average maintainer would normally have rejected this patchset after the first 5 to 10 bugs, depending on her/his patience. We chose instead to perform a thorough analysis of the full patchset, trying to identify typical patterns and anti-patterns of AI-generated kernel code.

## Goals

1. Document, through a concrete case study, the typical errors and structural problems found in an AI-generated kernel patchset.
2. Identify recurring patterns and anti-patterns of AI-generated code in the context of kernel development.
3. Derive a set of dos and don'ts (best practices) for using AI in the generation of kernel code and patchsets.
4. Contribute to the broader discussion on how the kernel development and review process should evolve to handle AI-generated submissions, given the strong asymmetry between generation effort and review effort.
