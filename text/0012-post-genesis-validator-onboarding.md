

# cep-0012:  Post Genesis Validator Onboarding

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0012](https://github.com/casperlabs/ceps/pull/0012)

Establishing a deep degree of trust in the security, performance & integrity of the Casper network is a necessary precursor to its success.  However for many entities, the failure to address AML/CFT concerns are a fundamental blocker to network participation.  A permission-less network, existing within the context of unregulated markets, attracts deep scepticism from those entities with significant AML.CFT related obligations.   Protocol mechanisms augmented by a form of off-chain social consensus, should be foreground such AML/CFT concerns.  Failure to do so will reduces risk appetite, and hence capital allocation, of prospective network investors (small & large).

Nowhere are the above facts more pertinent than when permitting new entities to onboard as network validators.  At genesis, the validator set is assumed to be of sufficient integrity so as to **safely** bootstrap the network, i.e. the emphasis at launch is correctness not throughput.  However post-genesis, the validator set will mutate at a fairly regular cadence in response to one or all of the following events: 

- new validators obtaining permission to join via an auction;
- honest validators voluntarily leaving;
- byzantine validators being ejected by the protocol.  

This CEP reviews the process of **securely** onboarding new validators.  It frames such a process within non-trivial security, infrastructure & token-economics considerations.  It focusses upon the following 3 use cases: 

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
- that the network's governance mechanisms revolve around off-chain social consensus.
- that it is in the interests of all network participants to adopt a pro-active approach to network-wide risk management;

## Motivation

[motivation]: #motivation

On several levels it is imperative that the protocol safely permits the validator set to organically grow above and beyond the genesis validator set.  It is inconceivable that the network does not support such a feature at genesis, i.e. it is an absolute & non-negotiable requirement of a minimal viable main-net.  Thus the motivation is crystal clear, it is that of a **demonstrably** healthy network.  Healthy from an operational, crypto-economic & (by extension) reputational perspective.  

The set of use cases this CEP touches upon are many & diverse:

1.  The reputation of the network must be such that (new) validator's are willing to continue to assume non-trivial capital risk in order to participate;

2.  The ratio of staked to non-staked token supply has game theoretic dynamics & hence real consequences;  

3.  The inability of the token holder base to secure an ambient ROI (via inflationary rewards) results in a slow drift towards centralisation;

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The proposal revolves around the mechanics of onboarding within a social consensus setting and describes several mechanisms that validators may wish to take advantage of so as to onboard in a manner befitting a potentially multi-billion dollar network.  

### 1. Establish a Casper Validator Association (CVA)

The risk aspects of running running a validator node revolves around the following (non-exhaustive) elements:

- Existential issues with the node software - e.g. a hitherto undiagnosed fault;
- Operational issues with the node deployment - e.g. a concerted DDOS attack;
- Jurisdictional issues associated with the business context - e.g. differing litigation risk profiles.
- Reputational issues within the social consensus context - e.g. cartel formation.

A rational risk mitigation strategy dictates that validators collaborate with each other.  Timely information shared over well defined communication channels is one such example of collaboration.  Validators operating in isolation are effectively disempowered when compared to ad-hoc groups of validators acting in unison.  However such ad-hoc groups tend to coalesce into political cliques, which is problematic as cliques typically focus upon narrow tactical objectives at the expense of **coherent** strategic objectives.  It is the contention of this CEP author that such a landscape is unfit for the purpose of nurturing and stewarding a successful crypto-economic network.

It is thus recommended that an association of Casper Validator Association is formally established, preferably in Switzerland.  Such an association will be endowed with reputational credibility and (a degree of) financial capital.  All genesis validators will be **required** to be members of this association.  The overall long-term aim is for the association to become one pillar upon which a future on-chain governance strategy can be implemented.  

Association membership incurs both duties & responsibilities.  To focus the minds of association members the initial objectives will be to:

- draw up legally binding articles of association;
- draft an initial constitution;
- establish a VDAO (see below);
- formulate a certification program (see below);

### 2. Establish a VDAO - i.e. validator DAO 

To reinforce the collaborative aspects of the validator association it is proposed to consider designing a validator DAO, aka a VDAO.  Such a VDAO will be a set of smart contracts permitting validators to confidentially submit either proofs or commitments.  Submitted information would be visible to other association members and would thus act as a trust amplifier.   

Proof submissions would revolve around legally binding documents addressing KYC, compliance & regulatory concerns.  The objective of these submissions is to reinforce the notion that the network is AML/CFT intolerant.  The legally binding aspect of such submissions within the context of a volunatry association is admittedly somewhat problematic, however registering the association as a Swiss entity might be helpful in this respect.

Commitment submissions are less black & white than proof submissions.  For example one might draft a commitment such as, "Members of this association will not support network activities that skew market pricing such as wash trading, front-running ...etc."  Such commitments aim to nudge the validators towards a shared set of values and goals that are clearly signalled to all network participants.  Entities engaging in such activities will thus be duly forewarned that the network operators, whilst perhaps unable to prevent such activities, will not tolerate them if and when proven to occur.  Thus commitments may act as deterrents.  

In due course, participation in this VDAO could be linked to the network's on-chain crypto-economic model.  I.E. the VDAO will act as an on-chain record of the degree to which a validator self-identifies & aligns with the network's integrity.  This information could be leveraged by the Proof-of-Stake rewards distribution scheme.  

### 3. Establish a validator certification program

A validator certification program will seek to impart the values underpinning the network's identity whilst detailing the duties & responsibilities expected of a validator during the course of their active engagement with the network.  It will revolve around a training program combined with a series of community inductions.  

The operational component of the training program will focus upon the mechanics of securely and safely operating a validator node.  The reputational component will focus upon participation within the wider social consensus process.  The technical component will focus upon imparting information regarding node software design & behavioural analysis.  The governance component will focus upon detailing the range of potential scenarios in which governance mechanisms impact upon operational concerns and how to respond to those scenarios in a timely manner.  

The certification program would be available online.  The content might be a mix of materials augmented by a set of relatively simple tests to ensure that the materials were at least reviewed.  Successful completion of the certification program would be recorded and placed upon a public record, perhaps via the VDAO outlined above.  

### 4. Partner with cold wallet providers

In order to onboard a prospective validator must by necessity obtain a sufficient amount of the network's token so as to successfully participate in the on-chain validator slot auction process.  Obtaining a sufficient amount of the network token requires participating as a buyer at one or more market venues such as exchanges, OTC desks and/or brokerages.  The end result of such participation is a **high-value** account upon the Casper network.

Such an account is by definition extremely sensitive from a security perspective.  Both access to and usage of the account will be highly constrained and must be auditable.  A so-called cold wallet would be the natural means via which the account would be managed.  Ideally such a wallet would support:

- multi-signature authorisation;
- off-line transaction signing;
- secure hardware devices;
- full & simplified auditing;

Developing this feature set is a non-trivial task and one best performed in collaboration with specialist hardware wallet providers.  It is hereby proposed that, as a matter of high priority, Casper Labs identifies and reaches out to hardware wallet providers of high repute within the space.  It could do this in collaboration with early adopter validators.  This feature set should be made available to both Genesis and prospective validators at the launch of main net. 

### 5. Enhance validator key management features

The Casper network has a few nice features in respect of account management.  These features could be leveraged and perhaps augmented so as to provide a more sophisticated validator experience in respect of PoS auction participation and node operation.  An enhanced experience becomes increasingly relevant for validator's operating potentially several nodes.

According to the technical documentation an account has a set of **weighted associated keys** to which the account owner can delegate deploy signing permissions.  The weights are used to derive action thresholds - when the weights of the key(s) used to preform an action exceed the action threshold then the action is execution.  Currently there are 2 supported action types: deploys & account recovery.

It is hereby proposed to add a 3rd action type, namely '**submit auction bid**'.  This will enable a validator to decouple the account key from the PoS auction participation key.  Mapping keys to well defined roles is best practise in respect of key SecOps.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

From a design perspective the above proposals require a blend of both soft and hard design.  By soft design is meant clarification of the off-chain operational processes that a validator will be expected to follow.  On the other hand hard design refers to core Casper software artefacts that may need to be either implemented or enhanced in support of validator onboarding.   

### Establishment of a Validator's Association

This is considered primarily a soft design problem.  It assumes that the proposal would be accepted by prospective Genesis set validators - this is actually by no means a given.  The design solution necessarily combines a diverse skillset: 

- legal expertise;
- community building;
- smart contract development (see VDAO);
- long-term thinking;

Successful implementation will require clear signals from the CasperLabs management team being transmitted to the core set of validators helping to incubate the network.  If these stakeholders can relatively quickly arrive at consensus on both the broad and finer strokes of the proposal then the association  may be established in a timely manner, i.e. before main-net.  It may even be possible to establish an embryonic VDAO before main-net but more likely this would be released sometime after main-net launch. 

### Partnering with hardware wallet providers

This requires engagement at the business level followed by commitment of relevant technically minded human resources.  The business level engagement should be relatively straightforward as the interests of all stakeholders are quite clearly aligned.  The key will be to identify and engage with wallet providers of repute as this in itself signals seriousness of intent.

The technical level engagement will require allocation of individuals comfortable navigating the landscape of blockchain cryptography.  Whilst most hardware wallets support multiple ECC key types, it must be ensured that all ECC key types supported by the Casper network are also supported by the wallet provider.  For secp256k1 and ed25519 this should not be a problem, however if sr25519 is to be adopted then there may need to be deeper collaboration with the wallet provider. 

The hardware wallet software may need to be updated so as to be able to securely dispatch signed transactions to the Casper network.  If enhanced key management features in support of validator onboarding are incorporated into the core Casper node software then the hardware wallet provider(s) will require very clear explanations of the programmatic mechanics of doing so.  
 
## Drawbacks

[drawbacks]: #drawbacks

1. The proposals outlined in this CEP are potentially problematic from a political & jurisdictional perspective.  The crypto space is unique in that attracts a fair number of non-regulated entities, such entities may very likely be put off by the idea of mandatory association membership.

2. The notion that validators may be classified in some respect, e.g. regulated | certified | other, requires some form of arbiter.  This is problematic if, for example, the arbiter is found to be byzantine.  

3. Any modifications that impact account key management are non-trivial and hence problematic with respect to the relatively aggressive main-net release schedule.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

In respect of establishing a validator association this is considered to be a partial formalisation of the hitherto proposed 'social consensus'.  It is the assertion of this CEP author that unless AML/CFT concerns are seen to be addressed, then the amount of capital and type of entities committed to the network will be limited in depth and scope.  This CEP also proposes strengthening key management narratives which is simple common sense given the amount of value potentially associated with validator accounts.

The impact of not forming an association is that political cliques will inevitably form.  Such cliques focus upon short-medium term tactical objectives to the detriment of coherent long-term strategic objectives.  Furthermore those isolated from cliques, possibly due to simple issues regarding language & culture, may feel somewhat alienated and hence less committed to participating in the network over the long term. 

The impact of not strengthening key management narratives is to take the risk that validator accounts are compromised due to the adoption of ad-hoc bespoke key management solutions.  Whilst such ad-hoc solutions are perhaps inevitable, partnering with specialist 3rd party wallet providers is a sensible proactive SecOps strategy.   

## Prior art

[prior-art]: #prior-art

In respect of a validator association:

- Due to it's potential reach, the Libra project has been forced to think deeply upon the regulatory setting within which the network operators conduct business.  
- The Tezos network has a strong validator community that has effectively self organised over time.

In respect of enhanced validator key management:

- The Polkadot network is notable in this regard.  It's account model has been adapted to support the notion of sub-keys being controlled by a master key.  The sub-keys can be used within the context of validator onboarding.
- Various hardware ledgers are focussing upon providing support for Proof-of-Stake networks.

## Future possibilities

[future-possibilities]: #future-possibilities

1.  Freeze accounts **provably** tainted to be associated with criminal activity.  The network's core account model could be updated to include a 'reject tainted accounts' setting.  If set to true then the network would implicitly reject transactions associated with tainted accounts.

2. Adopt ECC Key Algorithms with in-built multi-sig support.  This would bring the Casper network in line with other recently launched networks.