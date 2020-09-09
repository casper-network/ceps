

# cep-0012:  Casper Validators Association

## TLDR;

A new legal entity, the Casper Validator Association (CVA), will:

-  foreground Anti-Money Laundering (AML) and Combating the Financing of Terrorism (CFT) concerns;
-  play a pivotal role in the network's social consensus co-ordination mechanism;
-  act as the conduit through which new validators are on-boarded.

As a postscript the reader is reminded that an associated CEP will outline key management strengthening proposals.  By extension, these proposals impact upon the operational mechanics of post genesis validator onboarding.

## Introduction

[summary]: #summary

CEP PR: [casperlabs/ceps#0012](https://github.com/casperlabs/ceps/pull/0012)

Establishing deep trust in the integrity of the Casper network is a precursor to its success.  However a permission-less network, existing within the context of unregulated markets, attracts significant scepticism from institutional entities obliged to take AML/CFT concerns seriously.  This scepticism translates into reduced risk appetite and by extension reduced investment.  It is the assertion of the CEP author that on-chain mechanisms augmented by a form of off-chain 'social consensus', should foreground such AML/CFT concerns to the long-term benefit of the network as a whole.  

AML/CFT concerns apply particularly at the point of onboarding network validators, doubly so when a validator has accumulated sufficient stake weight via delegators.  At genesis, the validator set is assumed to be of sufficient integrity so as to **safely** bootstrap the network, i.e. the emphasis at launch is correctness not throughput.  However post-genesis, the validator set will mutate at a fairly regular cadence in response to one or all of the following events: 

- new validators obtaining permission to join via an auction;
- honest validators voluntarily leaving;
- byzantine validators being ejected by the protocol.  

This CEP considers the process of onboarding new validators.  It frames such a process within non-trivial security, infrastructure & token-economics considerations by focussing upon the following use cases: 

- a validator acting on the behalf of themselves;
- a validator acting on the behalf of themselves & delegators;
- a validator acting on behalf of delegators only.

The discussions assume the following:

- that addressing AML/CFT compliance concerns is in the long-term interest of the network;
- that the number of active validators is bounded by a protocol imposed ceiling;
- that byzantine validators are present, even at genesis;
- that whilst active markets exist for the network token, they are assumed to be mostly unregulated;
- that the network's crypto-economic security model is predicated upon so-called Proof-of-Stake (POS);
- that via a smart contract, the network exposes an auction mechanism permitting prospective new validators' to signal an intent to participate;
- that the network's governance mechanisms revolve around off-chain social consensus;
- that it is in the interests of all network participants to adopt a pro-active approach to network-wide risk management.
- that a validator may onboard as a result of attracting sufficient delegated stake weight - I.e. the validator's associated risk is based upon reputation rather than capital. 

## Motivation

[motivation]: #motivation

It is imperative that the Casper network validator set can safely evolve over time.  It is inconceivable that the network does not support such a feature at genesis, i.e. it is a non-negotiable requirement of a minimal viable main-net.  Thus the CEP motivation is crystal clear, it is a necessary step towards a **demonstrably** healthy network.  Healthy from an operational, crypto-economic & reputational perspective.  

A subset of the contextual use cases this CEP touches upon are:

1.  The reputation of the network must be such that (new) validator's are willing to continue to assume non-trivial capital risk in order to participate;

2.  The ratio of staked to non-staked token supply has game theoretic dynamics & hence real consequences;  

3.  The inability of the token holder base to secure an ambient ROI (via inflationary rewards) results in a slow drift towards centralisation;

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

### 1. Establish a Casper Validator Association (CVA)

Running a validator node entails significant risk.  Such risk revolves around the following elements:

- Existential issues with the node software - e.g. a hitherto undiagnosed fault;
- Operational issues with the node deployment - e.g. a concerted DDOS attack;
- Jurisdictional issues associated with the business context - e.g. new compliance regulations.
- Reputational issues within a social consensus context - e.g. cartel formation.

A rational risk mitigation strategy dictates that validators collaborate.  Timely information sharing over well defined communication channels is the backbone of such collaboration. 
Validators operating in isolation are disempowered as compared to collaborative ad-hoc groups of validators.  However ad-hoc groups tend to coalesce into political cliques focussing upon narrow tactical objectives at the expense of coherent strategic objectives.  This strategic incoherence is to the detriment of the network's long-term viability.

An ad-hoc landscape is unfit for the purpose of nurturing and stewarding a successful crypto-economic network.  It is thus recommended that an association, hereby called the Casper Validator Association, is formally established.  Such an association will be endowed with financial capital and reputational credibility.  It seems logical to incorporate the association in either Singapore or Zug, Switzerland.  

Association membership incurs both duties & responsibilities and all genesis validators will be required to be members of this association.  To focus the minds of association members the initial objectives will be to:

- draw up legally binding articles of association;
- draw up initial constitution;
- establish secure, auditable communication channels (see below);
- establish a VDAO (see below);
- establish a certification program (see below);

### 2. Establish secure, auditable communication channels

Validators need secure communication channels over which sensitive information can be transmitted and/or shared.  Such information may be sensitive in an operational sense (e.g. detailing a bug), or in a business sense (e.g. token movement disclosures).  Such information may originate from multiple sources simultaneously (e.g. errors showing up in logs) or from a single source in isolation (e.g. intent to leave the association).  

The choice of communication provider(s) is an important one.  Any choice needs to be:

- appropriate to the channel context;
- trusted by the validator community;
- available across differing OS platforms;
- available across nation-state firewalls;
- auditable in the event of disputes;

Social media platforms are public-domain and hence by definition auditable, however they are inappropriate for communication of security and/or price sensitive information.  For such 'internal' communications it is advised to setup a dedicated communications platform using open-source products such as matrix.

Auditabilty is a controversial subject that requires a form of mediation protocol.  I.E. a well-defined legally binding process must exist so that a mandated auditor can gain access to the subset of historical communication data pertinent to the  auditing context.  Initially this feature will probably not be supported as it will require consensus amongst validators following a probably protracted debate. 

### 3. Establish a VDAO - i.e. validator DAO 

To reinforce the collaborative aspects of the validator association it is proposed to consider designing a validator DAO, aka a VDAO.  Such a VDAO will be a set of smart contracts permitting validators to confidentially submit either **proofs** or **commitments**.  Submitted information would be visible to other association members and would thus act as a trust amplifier.   

Proof submissions would revolve around legally binding documents addressing KYC, compliance & regulatory concerns.  The objective of these submissions is to reinforce the notion that the network is AML/CFT intolerant.  The legally binding aspect of such submissions within the context of a volunatry association is admittedly somewhat problematic, however registering the association as a Swiss entity might be helpful in this respect.

Commitment submissions are less black & white than proof submissions.  For example one might draft a commitment such as, "Members of this association will not support network activities that skew market pricing such as wash trading, front-running ...etc."  Such commitments aim to nudge the validators towards a shared set of values and goals that are clearly signalled to all network participants.  Entities engaging in such activities will thus be duly forewarned that the network operators, whilst perhaps unable to prevent such activities, will not tolerate them if and when proven to occur.  Thus commitments may act as deterrents.  

In due course, participation in this VDAO could be linked to the network's on-chain crypto-economic model.  I.E. the VDAO will act as an on-chain record of the degree to which a validator self-identifies & aligns with the network's integrity.  This information could be leveraged by the Proof-of-Stake rewards distribution scheme.  

### 4. Establish a certification program

A validator certification program serves to impart the values underpinning the network's identity whilst detailing the duties & responsibilities expected of a validator during the course of their active engagement with the network.  It will revolve around a training program combined with a series of community inductions.

The operational component of the training program will focus upon the mechanics of securely and safely operating a validator node.  The reputational component will focus upon participation within the wider social consensus process.  The technical component will focus upon imparting information regarding node software design & behavioural analysis.  The governance component will focus upon detailing the range of potential scenarios in which governance mechanisms impact upon operational concerns and how to respond to those scenarios in a timely manner.  

The certification program would be available online.  The content might be a mix of materials augmented by a set of relatively simple tests to ensure that the materials were at least reviewed.  Successful completion of the certification program would be recorded and placed upon a public record, perhaps via the VDAO outlined above.  

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Establishment of a Casper Validator's Association plus associated legal, technical and communications infrastructure is considered primarily a soft design problem.  It assumes that the proposal would be accepted by prospective Genesis set validators - by no means a given.  The design solution necessarily combines a diverse skillset: 

- legal expertise;
- community building;
- smart contract development (see VDAO);
- long-term thinking;

Successful implementation will require clear signals from the CasperLabs management team being transmitted to the core set of validators assisting in the incubation of the network.  If stakeholders can relatively quickly arrive at consensus on both the broad and finer strokes of a proposal, then the association  may be established in a timely manner, i.e. before main-net.  It may even be possible to establish an embryonic VDAO before main-net but more likely this would be released at a later date. 
 
## Drawbacks

[drawbacks]: #drawbacks

1. The proposals outlined in this CEP are potentially problematic from a political & jurisdictional perspective.  The crypto space is unique in that attracts a fair number of non-regulated entities, such entities may very likely be put off by the idea of mandatory association membership.

2. The notion that validators may be classified in some respect, e.g. regulated | certified | other, requires some form of arbiter.  This is problematic if, for example, the arbiter is found to be byzantine.  

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

In respect of establishing a validator association this is considered to be a partial formalisation of the hitherto proposed 'social consensus'.  It is the assertion of the CEP author that not visibly addressing AML/CFT concerns will result in a reduction in both the amount of capital and prestige of entities committed to the network.  

The impact of not forming an association is that political cliques will inevitably form.  Such cliques focus upon short-medium term tactical objectives to the detriment of coherent long-term strategic objectives.  Furthermore those isolated from cliques, possibly due to simple issues regarding language & culture, may feel somewhat alienated and hence less committed to participating in the network over the long term. 

## Prior art

[prior-art]: #prior-art

- Due to it's potential reach, the Libra project has been forced to think deeply upon the regulatory setting within which the network operators conduct business.  It has formed [an association](https://libra.org/en-US/white-paper/#the-libra-association) in order to navigate a complex regulatory landscape.
- The Tezos network has a strong validator community that has effectively self organised over time.  

## Future possibilities

[future-possibilities]: #future-possibilities

1.  Freeze accounts **provably** tainted to be associated with criminal activity.  The network's core account model could be updated to include a 'reject tainted accounts' setting.  If set to true then the network would implicitly reject transactions associated with tainted accounts.
