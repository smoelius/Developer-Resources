This is the economic layer of the protocol. It incentivizes Task Givers, Solvers, Verifiers, Challengers, Referees, and Judges to work together in enabling trustless computation. A step-by-step breakdown of this protocol is presented below.


## Preprocessing steps
* Initialize TrueBit contract with a `submissionFee`, a `timeoutRate`, a `contemptRate`, and a universal tax rate `T`.
* Solvers and Verifiers that wish to participate in coming rounds deposit ETH to the TrueBit contract. They will only be able to participate in Tasks where their deposit, as set by Task Giver, is sufficient.

## The Main Algorithm

1. A Task Giver creates a Task on the TrueBit contract, by providing the following:
    * `task`: a computational task.
    * `timeOut`: a time value (probably in terms of blocks) to allow for performing of computation and waiting for a challenge.
    * `reward`: ETH which will be held in escrow by the contract.
    * `minDeposit`: the minimum deposit needed to participate as a Solver or Verifier.
      * Task Giver should require more skin-in-the-game for Solvers and Verifiers participating in tasks that have large economic downstream effects.

   Additionally, Task Giver must post an amount of ETH equal to `submissionFee` + `T` * d + `reward`.

2. Solvers who have the requisite `minDeposit` can bid to the TrueBit contract to take on the task, before `timeOut` elapses.
    * IF no solver takes on the task in the allotted time, THEN: Task Giver receives a refund minus `submissionFee`, and the protocol terminates.

3. Referees (i.e. the miners) select Solver by lottery.

4. The selected Solver privately computes task.
    * 4a. IF `timeOut` expires before a solution, THEN: Solver forfeits `timeoutRate` * `minDeposit` to the TrueBit contract, Task Giver receives a full refund, and the protocol terminates.

5. Solver commits the hashes of two solutions to the TrueBit contract, as well as the hash of a secret value `x`.  If `x` is even, then the first solution is the correct one; if `x` is odd, then the second solution is the correct one.

6. Before `timeOut` has elapsed, verifiers who have posted `minDeposit` can challenge one or both solutions by committing the hash of a secret value `y`.  (See the [whitepaper](http://people.cs.uchicago.edu/~teutsch/papers/truebit.pdf), page 41.)
    * 6a. IF no verifier challenges either solution, THEN the protocol proceeds to step 9.

7. Referees (i.e. the miners) select Verifier A by lottery.

8. Verifier A reveals `y`.
    * 8a. IF the hash of `y` does not match the one that Verifier A committed to in step 6, THEN: Verifier A forfeits `contemptRate` * `minDeposit` to the TrueBit contract, and the protocol returns to step 6.

9. Solver reveals `x` and the correct solution.
    * 9a. IF the hash of `x` does not match the one that Solver committed to in step 5, OR the hash of the revealed solution does not match the one determined by `x`, THEN: Solver forfeits `contemptRate` * `minDeposit` to the TrueBit contract, Task Giver receives a full refund, and the protocol terminates.

10. IF Verifier A challenged the correct solution (possibly in addition to challenging the incorrect solution), THEN Solver and Verifier A play a verification game.
    * 10a. IF Solver wins, THEN Solver receives Verifier A's `minDeposit`.
    * 10b. IF Verifier A wins, THEN: Verifier A receives Solver's `minDeposit`, Task Giver receives a full refund, and the protocol terminates.

11. IF no Verifier A was selected in step 6, OR Verifier A challenged just the incorrect solution, OR Solver won the verification game in step 10, THEN: before `timeOut` has elapsed, verifiers who have posted `minDeposit` can challenge the correct solution.
    * 11a. IF no verifier challenges the correct solution, **which should be the common case**, THEN the protocol proceeds to step 14.

12. Referees (i.e. the miners) select Verifier B by lottery.

13. Solver and Verifier B play a verification game.
    * 13a. IF Solver wins, THEN: Solver receives Verifier B's `minDeposit`, and the protocol returns to step 11.
    * 13b. IF Verifier B wins, THEN: Verifier B receives Solver's `minDeposit`, Task Giver receives a full refund, and the protocol terminates.

14. IF no Verifier A was selected in step 6, OR Verifier A challenged the correct solution (and lost, possibly in addition to challenging the incorrect solution), THEN: Solver receives the task reward ($), and the protocol terminates.

15. IF Verifier A challenged just the incorrect solution, **which should be the common case**, THEN:
    * 15a. IF hash of concat(`x`, `y`) is even, THEN Solver receives the task reward ($).
    * 15b. IF hash of concat(`x`, `y`) is odd, THEN Verifier A receives the task reward ($).

## Notes
* If Solver and Verifier A perform only correct computations, then they receive the task reward roughly equally as often.
* The main purpose of steps 11-13 is to thwart a malicious miner who tries to use their Solver and their Verifier A to get a bogus computation onto the blockchain.
* Steps 11-13 are in a loop because winning just one verification game is not sufficient to prove that a solution is correct.

## Values
* `submissionFee`: a small fee payed by Task Giver to cover TrueBit's gas costs.
* `timeoutRate`: the rate at which solvers are penalized for timing out.
* `contemptRate`: the rate at which solvers/verifiers are penalized for lying about what they have hashed.
* `T`: the universal tax rate, established when TrueBit contract is deployed.
* d: task difficulty as determined by `timeOut` provided by Task Giver.
