# Casper Payments: Gas Payments

## High level view
This is the first of a few CEPs on Casper Payment model. 
The motivation for this work is to make Casper much more user and dev 
friendly than it is now. There are a few topics here, that mix and 
a change to one of them might cause the domino effect on others, but we'll 
try to chunk them into logical units:

#### Token Standard 
Most of the system built on top of smart contracting platforms use tokens 
and it reasonable to put a lot of effort in designing good token standard 
or discover gaps in core functionalities that should be fixed before 
the Mainnet. The most popular token standard right now is ERC20. We have 
implemented its Casper version here https://github.com/casperlabs/erc20.
We think we can do better and offer our future community a token standard 
that avoids the "approve and call" pitfall. 
This work will be the subject of future CEP.

#### CSPR Implementation
One of the most important lessons we (as a blockchain community) learned 
from the Ethereum is that it's problematic when the platform's native token 
doesn't implement a good token standard. After our Token Standard is defined 
we will consider CSPR reimplementation to support the Standard.
This work will be the subject of future CEP.

#### Gas Payments
This part covers the topic of how users pay for the gas. It doesn't cover 
gas model itself. This work is a subject for current CEP.

#### Smart contract and accounts unification
The new model of our accounting/operational unit that should easily allow for 
all above. That means merging accounts and contracts (and urefs?) addressing 
spaces, enabling easy transfer of tokens between accounts and addresses, 
making uref sharing easy and ways of calling contracts.
This work will be the subject of future CEP.

## Gas Payments
In this section, we'll cover ways of paying for deploys that were discussed
in a past and we'll decide what we want to support.

### Self Payment
This is the simplest case. User should be able to use its own CSPRs to pay
for the execution of its own deploy (WASM or contract call).

### Delegeted Payment
Alice and Bob should be able to execute the following scenario:

1. Alice creates a Session Code (contract call) and signs it with her private key. 
2. Alice sends it to Bob.
3. Bob adds Payment Code information and signs it with his private key.
4. Bob sends it to the network.
5. Network executes the deploy in a way that:
    a. Session Code executes as Alice (her AccountHash is returned when calling `get_caller`),
    b. Payment Code executes as Bob, so he can pay for Alice's Session Code.
    
That enables Alice to interact with the network without her having to obtain CSPR tokens for gas.

### Contract Payment
It was considered may times to have a feature that allows Alice to call 
a contract that can pay for the execution without a need for her to have CSPR tokens.

To have that, the contract would have to perform some amount of computations 
to check if Alice is eligible for a free execution.

Unfortunately, this scenario is a subject of a spam attack, where Alice could 
force the network to perform a lot of computations by sending thousands of deploys.

The solution here is to support that feature on the host side and realized it on 
the protocol level (outside of Payment Code) on top of contract headers. 
The check whether Alice can execute the contract should have fixed cost acceptable 
for validators. Most of usecases that would use that feature can be realized with 
a mix of off-chain communication and Delegated Payment feature. Having that said 
we propose not to implement Contract Payment before the mainnet. It will be 
possible to add it later as an enhancement.

### Royalty Fee
Another idea that was considered was Royalirt Fee for smart contract owner: Alice 
would have to pay an additional fee for using a contract that would go to the contract owner. 

This feature is easy to implement on the smart contract layer inside the contract,
so we see no need to add that feature to the system. Fees management inside
contracts are rich pieces of logic and it would require a very complex solution 
to avoid execution smart contracts for that part. 

### Removing Payment Code
If the above scenarios are accepted, no longer we need a separate Touring-complete 
code execution for gas payments. We suggest to remove Payment Code and realized 
payments in a form of Delegated Payment (Self Payment being a special case of it, 
where Alice pays for herself) and realized gas payment on the protocol level.

