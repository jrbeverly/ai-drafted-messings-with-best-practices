# AI-Generated Engineering Guidelines

> [!WARNING]
> **AI-authored:** This change was autonomously planned and implemented by an AI software factory from a human-authored specification, with possible subsequent human review or modification.

> [!WARNING]
> This experiment is effectively abandoned. The generated material is retained primarily as a research artifact.

An experiment in using AI research capabilities to mass-generate a structured collection of engineering guidelines and "best practices".

The idea came from seeking to establish a set of engineering practices for a large set of codebases (manyrepo style), in which an AI could research a broad set of technologies and engineering concerns, synthesize the findings into consistent documentation, and use the resulting corpus as a shared framework for making implementation decisions, with many minor adjustments driven by AI to enhance the codebases.

It generated a lot of documentation. It did not generate a useful engineering framework. Primary fault was with the methodology of approach.

## Notes

- Guidance repeatedly collapses into "could", "consider", "depending on requirements". Accurate enough. Not useful as a house standard.
- Documents locally coherent, globally weak. Little evidence decisions in one area constrain choices elsewhere.
- Generated from research + synthesis, not from building a representative system. Almost no practical validation that the recommendations compose well.
- Trade-offs described, rarely resolved. Lots of "A vs B"; very little "use A, accept X, because Y matters more here."
- Verbosity is excessive in many ways, too many headers, much of the docs feel like "I don't want to read this"
- Defaults under-specified. Too much preserved option space. Standards probably need one boring path + explicit escape hatches.
- Conventions generated too early. Missing the feedback that normally comes from implementing the same pattern repeatedly and finding where it fails.
- Specificity often stops at principle level. "Use DI appropriately", "consider performance", etc. Correct but not operational.
- Standards don't take advantage of ongoing explorations into forcing functions as a means of directing desired results
- Lot of pointless negative guidance (Do this, don't do this) that seems weak in practice.

---

Overall, the experiment seems to have confused documenting best practices with establishing engineering practices. Methodology in generation is the likely cause of the pain point/friction point. Large scale construction of systems, then improving them over time likely to yield better results.
