# Incentive Layer Notes

**Todo**: Integrate into dispute resolution notes on wiki.




## Task Market

A contract matching task requests to solvers and verifiers. 

***


The **Task** is the basic object around which the protocol revolves. Its properties are summarized below.



##### Task Properties



| Type  | Name | Description |
| -------- | -------- | -------- |
| `address`     |  `giver`     |  The address supplying the initialization and payment information for the task. |
| `address` | `solver` | The address elected to solve the task during the bidding phase.|
| `uint` | `reward` | The reward offered for the task's completion |
| `uint` | `minDeposit` | The minimum deposit required of prospective solvers and verifiers |
| `uint` | `base` | The base block from which the current timeout is measured. |
| `uint` | `committed` | The block at which solutions are first committed. |
| `uint` | `timeout` | The timeout period to apply across phases of the task. |
| `bytes32` | `init` | Keccak256 hash of the initial VM state. |
| `bytes32` | `soln` | Keccak256 hash of the terminal VM state. |
| `bytes32[2]` | `soln_hashes` | Keccak256 hashes of correct and incorrect. |
|  `bool` | `choice` | Selection among the above two hashes made by solver. |




***


The operation of the Task market is to support the lifecycle of a task, while managing a progressive jackpot used to pay out honest participants. This works as follows.

1. An account will post information about a task it wants to compute, putting up a reward for its solution. A small fraction of the reward is paid as *verification tax* to the jackpot. This account is called the **Giver**.
2. The first account to put up a sufficient stake is elected to solve the task. This account is called the **Solver**. Additional accounts that put up stake are eligible to 'challenge' the solver's submissions. These are called **Verifiers**.
3. The elected solver supplies both correct and incorrect solutions then designates one of them for submission.
4. If a verifier finds an error, they challenge the result, initiating a *verification game*. 
5. If no verifiers finds an error, the solver reveals the solution and whether or not a forced error is in effect. 


The operations of each role are summarized below.


| Method | Caller | Description |
| -------- | -------- | -------- |
| `post`     | Giver     |  Posts a task to the task market.     |
| `bid` | Solver | Places a bid on an open task. |
| `commit` | Solver | Commits correct and incorrect solutions to a task. |
| `designate` | Solver | Designate one of the two solutions as submission. |
| `reveal` | Solver | Reveal random bits, solution, and steps. |
| `challenge` | Verifier | Challenges the solution submitted by a solver. |





***






### Jackpot



To incentivize solvers and verifiers to perform their roles and provide the giver with correct results, a fraction of a task's reward is deposited into a 'jackpot'. This jackpot is to be paid out periodically, awarding only actively honest participants.



This is implemented first as a simple jackpot contract. The fallback function collects funds up to the jackpots maximum. The `payout` function then makes a payout of a fixed fraction of the jackpot balance. 


***


```js
// Jackpot.sol
pragma solidity ^0.4.17;

contract Jackpot {
    
    uint min;
    uint max;
    uint8 divisor;


    function Jackpot (
        uint _min,
        uint _max,
        uint8 _divisor) 
        public {
        min = _min;
        max = _max;
        divisor = _divisor;
    }
    
    
    function payout (address recipient) private returns (bool) {
        // Pay out a fixed fraction.
        recipient.transfer(this.balance/divisor);
        Payout(recipient, this.balance/divisor);
    }
    
    
    function isActive () public constant returns (bool) {
        // Don't fall short the minimum.    
        return this.balance >= min;
    }

    function () public payable {
        // Don't exceed the cap.
        require (this.balance + msg.value < max);
        Deposit(msg.sender, msg.value);
    }

    event Payout (address indexed to, uint amount);
    event Deposit (address indexed from, uint amount);

    
}

```

***

The Task Market contract can then inherit from this and define its usage.


***

```javascript=
pragma solidity ^0.4.17;

import "./Jackpot.sol";

contract TaskMarket is Jackpot {
    
    
    
    mapping (bytes32 => uint) tasks;
    mapping (address => uint) deposits;
    uint size;
    uint tax;
                                    
                                
    function post (bytes32 init, uint min_deposit) payable returns (bytes32 id) {
        // todo
    }
    function bid (bytes32 id) payable returns (bool success) {
        // todo
    }
    function commit (bytes32 soln_hashes) returns (bool success) {
        // todo
    }
    function reveal (bytes32 soln, uint steps ,bytes32 r) returns (bool) {
        // todo
    }
    
    
}

```

***


### Posting a Task

To post a task, the giver supplies, along with the `init` hash and a `min_deposit`, a quantity of Wei, from which a fraction is used as 'verification tax', with the remainder offered to the solver as reward.

```javascript=
function post (
    bytes32 init,
    uint min_deposit
) payable returns (bytes32 id) {
    
    uint reward = msg.value - tax; // this needs to be multilied by difficulty
    Task T = Task(msg.sender, bytes32 init, uint min_deposit, uint reward, block.number);
    bytes32 hash = keccak256(T.sender, T.init);
    require (tasks[hash].giver == 0);                                
    tasks[hash] = T;
    Post (msg.sender, hash, reward);
    return hash;
}


event Post (address indexed giver, bytes32 id, uint reward);

```

*** 

### Placing a Bid

To place a bid on a task, a solver supplies a suitable deposit, together with the hash of their private random bits $r$. 

```javascript=
function bid (
    bytes32 id,
    bytes rand_hash
) public payable returns (bool) {
    Task T = tasks[id];
    require (T.giver != 0 && 
             T.solver == 0 && 
             T.min_deposit <= msg.value);

                                 
    deposits[msg.sender] += msg.value;
    T.solver = msg.sender;
    T.rand_hash = rand_hash;
    T.base = block.number;
    Bid (msg.sender, msg.value);
    return true;
}

event Bid (address indexed bidder, uint amount);
```

***

### Committing Solutions

Before reaching a timeout, the solver must submit a pair of solutions.


```javascript=
function commit(bytes32 id, bytes32[2] soln_hashes) public returns (bool) {
    Task T = tasks[id];
    require (T.solver == msg.sender);
    if (block.number >= T.base + T.timeout) {
        T.giver.transfer(T.deposit);
        deposits[T.giver] -= T.deposit;      // Giver's deposit is refunded
        deposits[T.solver] += T.min_deposit; // Solver's forfeits deposit
        return false;
    }
                                                                           
    Commit(msg.sender, id, soln_hashes);
    T.soln_hashes = soln_hashes;
    T.committed = block.number;                                                                        
    return true;
}

event Commit (address indexed solver, bytes32 id, bytes32[2] soln_hashes);
```
When this transaction is mined, including `T.soln_hashes` in a block, the solver compares his private random bits `r` with the hash of the block at height `T.committed`. The resulting decision comes in the form of a call to `designate` one of the solutions as the submission.

```javascript=
function designate (
    bytes32 id, 
     bool choice
    ) public returns (bool) {
    
    Task T = tasks[id];
    require (T.solver == msg.sender && 
             T.committed <= block.number &&
             ()); // need to check if this has already been called
    T.choice = choice;
    T.base = block.number;
}
```


### Revealing Solutions

// todo

### Challenging a Solution

// todo