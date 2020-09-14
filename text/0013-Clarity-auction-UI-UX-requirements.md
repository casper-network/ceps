# Auction UI/UX requirements for Clarity

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0013](https://github.com/CasperLabs/ceps/pull/13)

Clarity is expected to be the main conduit for user interaction with the auction system. This proposal suggests a number of basic monitoring and interactive workflows, with corresponding UI elements, that users should have available at mainnet launch.

## Motivation

[motivation]: #motivation

We anticipate that there would be at least 4 groups of users who may want to interact with the auction system

- Validators, delegators & prospective validators
- Traders & investors
- Researchers (including the CasperLabs economics team)
- Blockchain enthusiasts

It is natural that we should support auction monitoring and interaction workflows for validators, since their nodes form the backbone of the platform. However, we should anticipate the needs of the other groups. 

Traders must have easy means of monitoring the payoffs the platform, which are intimately connected to the auction system, both logically and in the software itself, to encourage correct price formation in the markets. 

Researchers, including the author of the CEP, expect an easily accessible high-level view of platform operation, enabling them to quickly discover unexpected behaviors that need additional consideration in our design and modeling processes. 

Finally, blockchain enthusiasts need to see this critical part of the platform work in real time, using a simple UI, both as an assurance of the correct operation of the platform and as a functional advertisement for the innovation Casper brings to the table.

## Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

We will describe the proposed additions to Clarity from a functional perspective first, providing sample workflows corresponding to each user group in the preceding motivations narrative. The identified protagonist should not be assumed to be the only type of interest in a given scenario (e.g., delegators manage withdrawal purses, too).

We will denote UI elements in *bold* and _italic_ for buttons and display elements, respectively. Certain workflows will require the protagonist to be logged into the CasperLabs signer, assumed to provide key management capabilities similar to Metamask. We will also references "pages," which should be imagined as distinct sets of UI elements, such as Main, Auctions, Analytics and Purses.

### Prospective validator - bidding

A prospective validator needs to *bid* to become one.

    1. Log into the CasperLabs signer.
    2. Navigate to the Auctions page.
    3. Submit a *bid,* inducing Clarity to generate a transaction calling into the relevan auction contract entry point with specified parameters.
    4. Receive confirmation of succesful entry of bid upon finalization, displaying parameters entered and current position in the auction.

### Prospective validator - monitoring the competition

A prospective validator with an entered bid ought to be able to monitor current state of the auction. 

    1. Stay on Main page and observe historic bid display and/or ranked list of present bids, with indication of presently staked validators with both stakes and bids.

### Validator - notification of possible loss of validator status

Presently staked validators ought to be notified that they are due to lose validator status due to an insufficient bid.

    1. Log into the CasperLabs signer.
    2. Stay on Main page and observe a warning.
        2b. If an auction already caused a validator to lose their status in some future era, first time log-in after the event ought to display a different, additional warning to this effect.

### Validators and delegators - purse management

Both validators and delegators need a seamless way to manage multiple withdrawal purses, in addition to a single reward purse.

    1. Log into the CasperLabs signer.
    2. Navigate to the Purses page.
    3. Monitor accumulated tokens in the reward purse and tokens in withdrawal purses by unlock time.
    4. Transfer fully unlocked withdrawn tokens and reward tokens as desired.
    
### Validators and delegators - slashing notification

### Delegator - choose a validator and delegate tokens

Delegators need to be able to pick their validators and prospective validators in an intelligent manner.

    1. Log into the CasperLabs signer (optional).
    2. Stay on Main page and observe historic bid display and/or ranked list of present bids (see one of the scenarios above), with indication of bidders' delegation rates and current amount of delegated tokens.
    3. Navigate to the Auctions page to delegate and/or undelegate (if signed in).

### Trader - monitoring rewards

Entities and agents trading Casper tokens on various exchanges must be able to observe the aspects of platform state immediately relevant to revenues.

    1. Stay on Main page and observe historic bid display and/or ranked list of present bids (see one of the scenarios above), with cumulative rewards and transaction fees that accrued to each validators, as well as per month values for the same.
    2. Stay on Main page and observe historic cumulative rewards display, including information on total extant token mass and per-period (month) accrual of rewards and transaction fees for the entire platform.
    3. Stay on Main page and observe historic display of cumulative and per period (month) token transfers.

### Researcher - download past history of bids & delegations

Researchers must be able to access basic data on the platform without having to write complex scripts.

    1. Navigate to Analytics page.
    2. Download underlying historic data for displays on the Main and other pages.

### Blockchain enthusiasts and researchers - exploring validator history

Blockchain fans, independent contributors, writers and others should be able to track histories of validators on the platform.

    1. Stay on Main page and click on a validator or prospective validator present on any display.
    2. Observe history of bids, delegations, rewards and whatever other data can be easily extracted.
    3. Navigate to Analytics page and download underlying data for any selected set of validators or prospective validators.

## Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

It is expected that there would be a UI element (button) and dialogs replicating the functionality for every publicly accessible entry point of the Auction, Mint and PoS contracts. This will be expanded upon in this CEP after review of present implementation of these contracts, which has some deviations from the (currently non-public) spec.

### Pages

Pages are intended to be groupings of discrete UI elements, not necessarily separate HTML documents.

#### Main

The entry point, housing all the monitoring features, such as present state of the auction.

#### Auctions

Houses publicly interactive entry points of the Auction contract. 

#### Analytics

Houses options for downloading various datasets corresponding to the data displayed on Main. The functionality here is expected to be unconnected to the Auction, Mint or PoS system contract functionality, but rather operating by means of queries or access to pre-fetched data that drives the Main page displays.

#### Purses

Houses purse display and management functionality. Requires log-in to access.

## Drawbacks

[drawbacks]: #drawbacks

In principle, every workflow presented in this CEP, or enabled by the UI elements we describe, can be replicated using a bash script and a client CLI. With growth of the platform, these would enable development of auction tooling that would enable many of the workflows we explicitly describe and implicitly enable in this CEP. However, we expect DevDAO to handle implementation, so, in effect, Clarity can simply *be* the tooling that, otherwise, would take time to come into existence as the community struggles to coordinate on first complex third party additions to the ecosystem. This leaves the potential opportunity costs of dedicating early stage ecosystem development resources managed by DevDAO to Clarity improvements as the only basis for a strong argument against this CEP. Commenters are encouraged to flesh out such arguments.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

The rationale for these suggested additions to Clarity is enhanced decentralization (through removing the barrier of interacting with the CLI). The CLI would be the primary alternative (other options may include whatver APIs we implement) and likely the primary choice for users with significant business on the platform.

## Prior art

[prior-art]: #prior-art

We do not believe there is, at present, a comparable web interface implemented for a similar system. While eBay would appear to be an example, its UI appears to have remained in stasis for over a decade and does not provide much inspiration. To the author's knowledge, based on limited prior experience, the auction UI for search ad auctions, possibly a type of auction with the greatest number of participants of any type of auction, is rather utilitarian and provides only limited means of monitoring the results of the auctions.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

We expect that the notion that these features have practical utility, in the presence of the CLI, will receive some pushback during the development of this CEP. Additionally, as the author is not a UI designer, we expect critical feedback about the implicit layout envisioned in this CEP.

## Future possibilities

[future-possibilities]: #future-possibilities

We believe that this feature should be considered a self-contained, one-time addition, until specific business requirements are presented for possible extensions.
