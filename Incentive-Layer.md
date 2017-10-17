# The Incentive Layer

This is the economic layer of the protocol. It incentivizes Task Givers, Solvers, Verifiers, Challengers, Referees, and Judges to work together in enabling trustless computation. A step-by-step breakdown of this protocol is presented below.


## Preprocessing steps
* Initialize Jackpot contract with a large deposit from a philanthropist.
* Initialize TrueBit contract with a universal `tax rate T`.
* Solvers and Verifiers that wish to participate in coming rounds deposit ETH to the Truebit contract. They will only be able to participate in Tasks where their deposit, as set by the Task Giver, is sufficient.
* Solvers generate `private random bits r` and commit their hash to the Truebit smart contract.

## The Main Algorithm

1. a Task Giver creates a Task on the TrueBit contract, by providing the following:
  * `task`: a computational task (or more accurately, its hash).
  * `timeOut`: a time value (probably in terms of blocks) to allow for performing of computation and waiting for a challenge.
  * `reward`: ETH which will be held in escrow by the contract ([more details](#rewards))
  * `minDeposit`: the minimum deposit needed to participate as a Solver or Verifier. 
    * The Task Giver should require more skin-in-the-game for Solvers and Verifiers participating in tasks that have large economic downstream effects.

2. Solvers who have the requisite `minDeposit` AND `random bits` can bid to TrueBit smart contract to take on the task, before `timeOut` elapses.

3. Referees (i.e. the miners) select one Solver by lottery.
  * IF no Solver takes on the task in the allotted time, THEN the task request is canceled and Task Gives receives a full refund.

4. The selected Solver privately computes task. 
  * 4a. IF `timeOut` expires before a solution, THEN the Solver forfeits his deposit to the jackpot and the protocol terminates.
  * 4b. ELSE, the Solver commits both a "correct" and "incorrect" solution to the TrueBit contract (more accurately, the hash of a solution).

5. The next block gets mined. Depending on the `block hash` and their previously committed-to `random bit r`, the Solver knows whether a forced error is in effect. Therefore, the Solver pings the Truebit contract and designates one of their submitted hashes as the solution.

6. Before `timeOut` has elapsed: verifiers who have posted `minDeposit` can challenge (the hash of) solution. They do this by committing the hash of an even integer to the Truebit contract to commit to a challenge.

7. After `timeOut` elapses, interested Verifiers broadcasts to the blockchain this hashed number in the clear to reveal their action.
  * 7a. IF no Verifier challenges solution, THEN:
    * 7aa. Solver reveals `r`. IF there was no forced error, THEN Solver reveals solution and received the task reward ($). (This is the most common play-out of the game).
    * 7ab. Solver reveals `r`. IF there was a forced error, THEN protocol has failed since no one challenged. (This is the most uncommon play-out of the game)
  * 7b. ELSE a Verifier challenges solution, THEN: the Solver reveals their `random string r` to the Truebit contract. Referees check it against their commitment from the preprocessing step.
    * 7ba. IF hash of concat of r and block hash is small (as determined by the forced error rate), THEN a forced error is in effect.
      * Note: in these cases, the Solver does not receive a reward, but instead a Jackpot payout much greater than reward. This is because we assume they would have challenged their own solution.
      * 1. Solver reveals their secondary (aka "correct") solution in the clear.
      * 2. IF no Verifier challenges Solver's secondary solution before `timeOut`, THEN Verifier wins a fraction of the `jackpot J`, scaled for task difficulty.
        * -> Note: If there are multiple Challengers with same solution hash, they split the Jackpot reward in exponentially-decreasing chunks.
        * -> Note: If there are multiple distinct challenges, then they play the verification game serially. (Challenger ordering will be FIFO.)
      * 3. ELSE a Verifier challenges Solver's secondary solution, THEN they play the verification game.
        * -> IF Solver loses, THEN the Verifier penalties, Verifier rewards, Solver penalties, and refunds to the Task Giver are the same as next step.
        * -> ELSE IF Solver wins (i.e. their secondary solution was correct), the original Verifier gets their jackpot reward.
    * 7bb. ELSE, the error was not forced.
      * 1. Solver reveals solution in the clear, and the Solver and Verifier play a verification game.
        * In case of challenges from multiple verifiers, verification games play in serial, until a Challenger defeats Solver or the Solver defeats all Challengers. 
        * 1a IF solver wins, the Verifier forfeits half of their deposit to the Jackpot and the other half to the Solver.
          * Note: This is not a large reward, compared to winning a jackpot payout for an unforced error.
        * 1b. ELSE IF Challenger wins, the Solver pays back the Task Giver reward and tax, pays at most half of his deposit to Verifier (exponentially-decreasing given how many challenges there were), and forfeits remaining funds to the `Jackpot`. (the Task Giver still doesn’t know the correct solution in this case. The task needs to be re-issued.)

## Notes on Economics
* The jackpot payout for discovering a forced error must cover amortized payment for all verification tasks. (i.e. have a worthwhile Expected Value).
* I.e.: (Jackpot payout) >= (fair compensation for one task) * 1 / (forced error rate).
* The jackpot for each task scales proportionally with the task’s complexity (to equally incentivize work on both easy and hard tasks).
* The max jackpot payout for forced error is 1/3 total jackpot size. (To prevent the Jackpot from getting depleted).

## Rewards
* reward >= (cash equivalent of task difficulty d based on the timeOut) + (total tax of T * d).
* This has to cover:
  1. cost of computation done by Solver
  2. payment for Verifiers. This goes into Jackpot for now as a Tax, and gets paid out to the one lucky verifier who finds a forced-error in the future and amortizes all past costs.
  3. work done by referees and judges (i.e. Ethereum gas fees of future rounds).

## Values
* T: the universal tax rate, established when Truebit contract is deployed.
* d: task difficulty as determined by timeOut provided by Task Giver.