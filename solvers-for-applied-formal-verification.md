# Solvers for Applied Formal Verification

{% embed url="https://github.com/serial-garden/solvers" %}

Solvers and formal verification tools often have brittle coupling that makes managing version interoperability difficult and painful. Even [theoretically compatible](https://semver.org/) upgrades can still suffer significant unexpected performance regressions, placing a heavy burden on upstream contributors to shield users from a deteriorated user experience.

One strategy for alleviating this thrash could be a [**rolling-release distribution of common SMT and SAT solvers**](https://github.com/serial-garden/solvers). Checking configuration candidates against dependent projects ahead of time for functional and performance regressions minimizes the chances of new releases breaking dependent projects.

The scope of this effort includes serving a larger interest in studying automation of software availability and compatibility to a wide variety of distribution targets, including OS package managers and container images. It should be possible to alleviate the need for constant manual intervention from the release and publishing processes.

