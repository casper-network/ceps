

# cep-0008:  Post Genesis Validator Onboarding

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0008](https://github.com/casperlabs/ceps/pull/0008)

At genesis, it is assumed that a validator set of sufficient integrity will exist so as to **safely** bootstrap the network, i.e. the emphasis upon launch is correctness not throughput.  Post-genesis, the active validator set will mutate in response to one or all of the following events: 

- new validators obtaining permission to join via an auction;
- honest validators voluntarily leaving;
- byzantine validators being ejected by the protocol.  

This CEP focusses upon the process of **securely** onboarding new validators.  It frames such a process within non-trivial security, infrastructure, engagement & token contextual considerations.  In particular it focusses upon the following 3 use cases: 

- a validator acting on the behalf of themselves;
- a validator acting on the behalf of themselves & delegators;
- a validator acting on behalf of delegators only.

The discussions assume the following:

- that addressing compliance concerns such as Anti-Money Laundering (AML) or Combating the Financing of Terrorism (CFT) is in the long-term interest of the network.
- that the number of validators within the set is bounded by a protocol imposed ceiling, such a ceiling seeks to find a counterbalance between network participation & performance;
- that byzantine validators are present, even at genesis;
- that whilst active markets exist for the network token, they are assumed to be unregulated.
- that the network's crypto-economic security model is predicated upon so-called Proof-of-Stake (POS);
- that via a smart contract, the network exposes an auction mechanism permitting prospective new validators' to signal an intent to participate.
- that the network's governance mechanisms revolve around some form of so-called social consensus.
- that it is in the interests of all network participants to adopt a pro-active approach to network-wide risk management;

## Motivation

[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

On several levels it is absolutely essential that the protocol safely permits the validator set to organically grow above and beyond the genesis validator set.  It is inconceivable that the network does not support such a feature at genesis, i.e. it is an absolute & non-negotiable requirement of a minimal main-net.  Thus the motivation is crystal clear, it is that of a **demonstrably** healthy network.  Healthy from an operational, crypto-economic & (by extension) reputational perspective.  

The set of use cases this CEP touches upon are many & diverse:

1.  The reputation of the network must be such that (new) validator's are willing to continue to assume non-trivial risk in order to participate;

2.  The ratio of staked to non-staked token supply has game theoretic dynamics & hence consequences;  

3.  The inability of the token holder base to secure an ambient ROI (via inflationary rewards) results in a slow drift towards centralisation;

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The proposal has two aspects that the mechanics of onboarding within a social consensus setting.  

### 1. Formally establish & incentivise a validator association

In a mundane sense a validator is simply an operator running node software incubated by CasperLabs.  However the act of doing so incurs a **risk** burden, a risk burden laden with non-negligible financial consequences.  Such risk revolves around the following (non-exhaustive) elements:

- Existential issues with the node software - e.g. a hitherto undiagnosed fault;
- Operational issues with the node deployment - e.g. a concerted DDOS attack;
- Jurisdictional issues associated with the business context - e.g. differing litigation risk profiles.
- Reputational issues within the social consensus context - e.g. cartel formation.

In order to partially mitigate such risk, e.g. knowledge sharing, validators are effectively obliged to collaborate with each other.  Those acting in isolation are disempowered when compared to ad-hoc groups of validators acting in unison.  However such ad-hoc groups tend to form political cliques with **incoherent** strategic objectives.  Over time this landscape is unfit for the purpose of nurturing and stewarding a successful crypto-economic network.

It is thus recommended that an association of Casper network validators is formally established, preferably in Switzerland.  Such an association will be endowed with reputational credibility and a degree of financial capital.  The overall long-term aim is for the association to become one pillar of a future on-chain governance strategy.  Association membership incurs both duties & responsibilities, all genesis validators will be **required** to be members of this association.

To focus the minds of association members the initial objectives will be to:

- draw up legally binding articles of association;
- draft an initial constitution;
- designate a working committee to explore a DAO;
- formulate a certification program;

In respect of creating a DAO, such a DAO will be built upon a set of smart contracts that manages, amongst other things, association membership.  Ultimately this DAO could be linked to the network's crypto-economic model - e.g. unless one is a member then one cannot participate in the auction process.

### 2. Establish a validator certification program

Prospective validators will be offered a certification program.  Such a program will seek to impart the values underpinning the network's identity whilst detailing the duties and responsibilities expected of a validator during the course of their active engagement with the network.

It will revolve around a training program combined with a series of community inductions.  The operational component of the training program will focus upon the mechanics of securely and safely operating a validator node.  The reputational component will focus upon how to participate within the social consensus process.  

### 3. Partner with one or more hardware wallet providers

Develop a validator specific application 
 



It is proposed 

Explain the proposal as if it was already approved and implemented. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.

For implementation-oriented CEPs (e.g. for node internals), this section should focus on how other developers should think about the change, and give examples of its concrete impact. For policy CEPs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the CEP. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

[drawbacks]: #drawbacks

1. The proposals outlined in this CEP are potentially problematic from a political & jurisdictional perspective.  The crypto space is unique in that attracts a fair number of non-regulated entities, such entities may very likely be put off by the idea of mandatory association membership.
2. The notion that validators can be classified in some respect, e.g. regulated | certified | other, requires some form of arbiter.  This is problematic if, for example, the arbiter is found to be byzantine.  

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

A very important thing to list here is ideas that were discarded in the process, as these tend to crop up again after a while. Describing them here saves time as it allows people discussing those ideas again in the future to refer to this document.

## Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For development focused proposals: Does this feature exist in other applications and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your CEP with a fuller picture.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the CEP process before this gets merged?
- What related issues do you consider out of scope for this CEP that could be addressed in the future independently of the solution that comes out of this CEP?

## Future possibilities

[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the CEP you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future CEP; such notes should be in the section on motivation or rationale in this or subsequent CEPs. The section merely provides additional information.





##################################################################################################

delegate/undelegate/add_bid/withdraw_bid/run_auction

##################################################################################################