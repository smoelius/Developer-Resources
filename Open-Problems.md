
**In the case of a forced-error, how does the protocol incentivize verifiers to check the Solver's secondary (i.e. "correct") solution?**

_The scenario_:
* There is a forced error in effect. A verifier successfully challenges the Solver’s original (incorrect) solution. The solver then reveals their secondary (supposed-to-be correct) solution. What if this one is also wrong? How would a verifier who challenges this be compensated? Would they receive part of the jackpot? More thoughts below..

_Problems_:
* Why would any verifiers challenge this second solution? They already know it’s not a forced error, so they don’t have an incentive to solve.
* Some consolation: this is a highly unlikely case. The solver wouldn’t have known that there was going to be a forced error, so they would have computed their “correct” solution correctly, expecting that it would get challenged through normal protocol operation otherwise. -> But we still need to deal with the 1/1000 case.

_Potential Solutions_:
* Potential solution, albeit centralized: Truebit foundation could run verification for solutions after forced-error is found.
* Potential solution, but won’t work: giving part of the jackpot to second challenger.. They still don’t have incentive to calculate, because in the absence of “2nd-layer forced errors”, they never expect the solution to be wrong.
* Potential solution, seems plausible: the challengers have already computed the correct solution (because they were able to identify the forced error). They can use the same solution to challenge the Solver’s secondary solution. This involves zero marginal-cost, so seems like they would do it, even if there are minimal rewards to be had.

_Other notes_:
* Problem, although minor and won’t effect protocol: there is information leaking, as such: The solver posts both correct (SolutionA) and incorrect (SolutionB) solution hashes to the blockchain. They find there is a forced error in effect and select SolutionB as their submission. Verifiers compute and find that they disagree with the hash of SolutionB (i.e. they will challenge), but they also see that their solution agrees with the hash of SolutionA, the Solver’s alternate solution. As such, they also know that there is a forced error in effect, before the protocol has played out. This seems like it won’t actually effect how the protocol plays out, and is probably fine.

---

**A malicious party could potentially monitor the TaskBook and DoS the execution of certain tasks by continuously challenging.**

In the current protocol design, bogus challenges are disincentivized by requiring Challengers to post a `deposit`, which they lose in the case of an unsuccessful challenge.

This means that an adversary could continuously challenge a solution, making it run through time-intensive verification games, at a cost of `deposit` per challenge. This would be economical as long as the cost of the lost deposit is outweighed by downstream economic gains of the solution being delayed.

An example: the mob has taken someone hostage and is requiring $10m in payment through the Doge/Ethereum bridge in 24 hours (which relies on Truebit). Someone who wants to prevent the payment could simply watch the `TaskBook` and keep challenging the `Scrypt` verification until 24 hours runs out.

An analogous example could be when a smart contract requires a tx by a certain time (e.g. a financial contract). 

The core issue this exposes is timeliness. The protocol does not currently give any guarantees on how long a Task will take, even if a Solver is immediately available.

Potential solutions for this could be:
* Making consecutive challenges to the same Taks to become more expensive. This requires the protocol to take the "best" challenges first. A mechanism for determining this is to allow Challengers to "stake" their deposits behind challenge hashes.
* Requiring larger deposits. This would increase the cost to the attacker, but could also discourage hoenst participants.

---

**Does the TrueBit smart contract enforce a minimum `deposit` and `timeOut` value based on the `Task` difficulty?**

A `Task Giver` is responsible for setting a required `deposit` and `timeOut` when they create a `Task`. They need control over these values since "task difficulty" on its own may not fully encapsulate the economic cost of a task. For instance, verifying Scrypt for a Doge/Ethereum relay tx may be computationally simple; however, it might have large economic downstream effects. In this case, the Task Giver would want to require higher `deposit` values, to increase skin-in-the-game for to-be Solvers and Verifiers. The question is, should the smart contract enforce a _minimum_ `deposit` and `timeOut`?

---

**How do we account for total gas costs necessary to run a Task through the protocol?**

This needs to be accounted as part of the `reward` deposited by the Task Giver when creating a new Task. The reward needs to cover 1) the cost of computation by the Solver, 2) indirect payment for Verifiers (i.e. as Tax into the Jackpot, which will eventually be collected), and 3) work done by referees and judges (i.e. Ethereum gas fees to carry out the protocol).

---

**How is the Solver selected exactly?**

Is it simply who's tx comes in first (i.e. determined by miners)?

---

**In the case of a forced error, are jackpot payouts calibrated correctly to prevent a Solver from posing as multiple Verifiers?**

In the current protocol design, the Solver automatically receives part of the jackpot in these cases, but would they not be incentivized to go for even more of it?

I.e. Truebit participants could run a modified Truebit client, which has many Verifiers (i.e. with deposits made) ready and waiting for the moment that they become Solver and see a forced error. These Verifiers stay dormant and do not challenge any solutions. But as soon as the Solver notes a forced error, the client automatically gets them all to challenge, taking the max possible Jackpot.

(This referred to in section 5.3 on “flood of challengers”, but this talks about per-challenge-payout, whereas the participant is interested in their total payout as long as marginal reward > marginal cost?)

---

**In what order are challenges processed by the contract, in cases where there are multiple Challengers?**

Current solution is use a first-in first-out (FIFO) ordering.

---

**Data withholding**
* Great write up on wiki: https://github.com/TrueBitFoundation/webasm-solidity/wiki/Data-availability
* Essentially: TaskGiver posts a task hash to Truebit contract, but does not publish actual data for any solvers to find.
* Various attacks possible: TaskGiver trying to become Solver themselves, TaskGiver finding identity of verifiers and bribing them, etc.
* Possible solutions through use of Swarm, IPFS, erasure coding, etc.

---

**Solvers / verifiers need to store complete state of WASM machine at each step. How to keep these around for long running tasks?**

---

**WASM metering**
* Solver / verifier need to know an upper bound for the running time of task in order to properly incentivize and run the verification game fairly.

---

**Optimization: the off-chain interpreter needs to only send the part of the WASM state that is relevant to the next op to run (which is not necessarily the whole machine state).**
* for each WASM op, we know which parts of the state it can (potentially) change.
* "only the relevant state" means we use a merkle tree for the full state, only submit the single leaf of the merkle tree that is accessed plus a merkle proof for it.

---
